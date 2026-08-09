## 4. Evidências de Validação

### 4.1 Simulação

A imagem abaixo é do Questa (ModelSim), rodando o testbench `fp_adder_de10lite_vhd_tst` em cima do top-level já com os pinos da placa. Montamos o testbench para ser autoverificável. Ele aplica uma sequência de combinações de switches e pulsos de `KEY`, compara o resultado que sai do `fp_adder` com o valor esperado calculado à parte, e vai contando num sinal chamado `erros` toda vez que gera diferença. No fim da simulação, o sinal `fim_sim` sobe para `1`.

No print, o cursor está em 1440 ns, perto do fim da simulação. Nesse instante os dois sinais de controle do testbench mostram o que queríamos ver: `fim_sim = TRUE` e `erros = 0`. Todos os vetores de teste rodaram e nenhum resultado do `fp_adder` divergiu do valor esperado calculado à parte no testbench. Essa é uma verificação mais confiável do que apenas olhar a placa, pois o próprio testbench compara automaticamente cada resultado, em vez de uma pessoa conferir à mão caso por caso.

Podemos confirmar o mecanismo olhando a trilha do `KEY`, visto que ela fica alternando entre o valor `3` (`"11"`, os dois botões soltos, o estado parado entre um vetor e outro) e o valor `2` (`"10"`, ou seja `KEY(0) = '0'` pressionado) repetidas vezes ao longo da simulação, que é justamente o `KEY0` carregando um novo operando A antes de cada novo teste. Perto do fim aparece também um único pulso com o valor `1` (`"01"`, `KEY(1) = '0'`): o `KEY1` limpando o operando A, um vetor à parte só pra conferir que o clear funciona. Contando essas trocas junto com as mudanças na trilha de `SW`, podemos ver mais de dez configurações de entrada diferentes passando ao longo da simulação, cada uma seguida do pulso de `KEY` correspondente, o mesmo mecanismo que a gente testou na mão na Etapa 3, porém automatizado.

![Simulação no Questa: testbench autoverificável, erros = 0 ao final da execução](../doc/img/simulacao_testbench_autocheck.png)

### 4.2 Código VHDL Final

#### 4.2.1. Top-level adaptado para a DE10-Lite (`fp_adder_de10lite.vhd`)

Esse é o módulo de topo que realmente vira bitstream e roda na placa. Ele substitui o `fp_adder_test` do livro (Listing 3.20), pensado para uma placa antiga com 8 chaves, 4 botões e display multiplexado, que não é compatível com a DE10-Lite. Por isso quase tudo aqui foi reescrito; a única peça que entra sem alteração é o próprio `fp_adder`, chamado como componente.

```vhdl
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity fp_adder_de10lite is
   port(
      MAX10_CLK1_50 : in  std_logic;
      SW            : in  std_logic_vector(9 downto 0);
      KEY           : in  std_logic_vector(1 downto 0);
      LEDR          : out std_logic_vector(9 downto 0);
      HEX0          : out std_logic_vector(7 downto 0);
      HEX1          : out std_logic_vector(7 downto 0);
      HEX2          : out std_logic_vector(7 downto 0);
      HEX3          : out std_logic_vector(7 downto 0);
      HEX4          : out std_logic_vector(7 downto 0);
      HEX5          : out std_logic_vector(7 downto 0)
   );
end fp_adder_de10lite;

architecture arch of fp_adder_de10lite is

   signal signB : std_logic;
   signal expB  : std_logic_vector(3 downto 0);
   signal fracB : std_logic_vector(7 downto 0);

   signal signA : std_logic                    := '0';
   signal expA  : std_logic_vector(3 downto 0) := (others => '0');
   signal fracA : std_logic_vector(7 downto 0) := (others => '0');

   signal sign_out : std_logic;
   signal exp_out  : std_logic_vector(3 downto 0);
   signal frac_out : std_logic_vector(7 downto 0);

   signal zero_flag : std_logic;

   constant SSEG_MENOS   : std_logic_vector(7 downto 0) := "10111111";
   constant SSEG_APAGADO : std_logic_vector(7 downto 0) := "11111111";

begin

   signB <= SW(9);
   expB  <= SW(8 downto 5);
   fracB <= '1' & SW(4 downto 0) & "00";

   reg_operando_a : process(MAX10_CLK1_50)
   begin
      if rising_edge(MAX10_CLK1_50) then
         if KEY(1) = '0' then
            signA <= '0';
            expA  <= (others => '0');
            fracA <= (others => '0');
         elsif KEY(0) = '0' then
            signA <= signB;
            expA  <= expB;
            fracA <= fracB;
         end if;
      end if;
   end process;

   fp_add_unit : entity work.fp_adder
      port map (
         sign1    => signA,
         sign2    => signB,
         exp1     => expA,
         exp2     => expB,
         frac1    => fracA,
         frac2    => fracB,
         sign_out => sign_out,
         exp_out  => exp_out,
         frac_out => frac_out
      );

   zero_flag <= '1' when frac_out = "00000000" else '0';

   LEDR(9)          <= sign_out;
   LEDR(8)          <= zero_flag;
   LEDR(7 downto 0) <= frac_out;

   disp_exp_res : entity work.hex_to_sseg_de10
      port map (hex => exp_out, dp => '0', sseg => HEX2);

   disp_frac_hi : entity work.hex_to_sseg_de10
      port map (hex => frac_out(7 downto 4), dp => '0', sseg => HEX1);

   disp_frac_lo : entity work.hex_to_sseg_de10
      port map (hex => frac_out(3 downto 0), dp => '0', sseg => HEX0);

   HEX3 <= SSEG_MENOS when sign_out = '1' else SSEG_APAGADO;

   HEX4 <= SSEG_APAGADO;
   HEX5 <= SSEG_APAGADO;

end arch;
```

**Portas.** Trocamos os nomes genéricos do livro pelos nomes reais dos pinos da placa: `SW`, `KEY`, `HEX0` a `HEX5` e `LEDR` já são reconhecidos automaticamente pelo Quartus a partir do `.qsf` da DE10-Lite. Também entrou o `MAX10_CLK1_50`, que não existia na versão original. Precisamos dele porque passou a ter um elemento sequencial no circuito.

**Operando B.** É ligação direta das chaves: `SW(9)` vira o sinal, `SW(8 downto 5)` vira o expoente. A fração é montada concatenando um `1` fixo (para garantir que o número entra normalizado), os 5 switches que sobraram, e dois `0` no final. Restam só 5 dos 8 bits realmente controláveis, oferecendo menor precisão, mas é a limitação de ter apenas 10 chaves.

**Operando A e o registrador.** Essa é a mudança mais importante da etapa. Na primeira versão, A era uma constante fixa; só que com 10 switches não dá para controlar os 26 bits dos dois operandos ao mesmo tempo. A solução foi transformar A num registrador: aperta `KEY0` e ele copia o que está nas chaves naquele instante; aperta `KEY1` e ele zera. É o único trecho sequencial do projeto inteiro, uma vez que todo o resto do circuito é combinacional puro e reage na hora, sem depender do clock.

**A instância do `fp_adder`.** Aqui há um `port map`, ligando `signA/expA/fracA` e `signB/expB/fracB` nas entradas e recolhendo `sign_out/exp_out/frac_out` na saída. Não tem lógica nova nessa parte, é apenas o componente da Etapa 1 sendo chamado sem mudanças.

**LEDs de depuração.** `LEDR(9)` repete o sinal do resultado, `LEDR(8)` acende quando a fração resulta em zero (útil para não precisar ficar decorando que `0x00` é zero), e os outros 8 mostram a fração crua em binário, bit a bit, como ela sai do somador.

**Displays.** Cada instância de `hex_to_sseg_de10` recebe 4 bits e devolve o padrão de segmentos correspondente. `HEX2` fica com o expoente, `HEX1` e `HEX0` dividem a fração nos dois nibbles. `HEX3` é tratado à parte porque não precisa decodificar nada em hexadecimal: só acende o segmento do meio quando o resultado é negativo, e fica todo apagado quando é positivo. `HEX4` e `HEX5` sobraram sem uso depois que a professora pediu para tirar a memória do operando A da tela, então ficam apagados o tempo todo.

#### 4.2.2. Decodificador de 7 segmentos adaptado (`hex_to_sseg_de10.vhd`)

Esse módulo não muda de teste para teste: é somente a tabela-verdade que transforma 4 bits num padrão de 7 segmentos. A parte que precisou de atenção foi adaptar para a polaridade da DE10-Lite.

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity hex_to_sseg_de10 is
   port(
      hex  : in  std_logic_vector(3 downto 0);
      dp   : in  std_logic;
      sseg : out std_logic_vector(7 downto 0)
   );
end hex_to_sseg_de10;

architecture arch of hex_to_sseg_de10 is
   signal seg : std_logic_vector(6 downto 0);  -- ordem: g f e d c b a
begin

   with hex select
      seg <= "0111111" when "0000",   -- 0
             "0000110" when "0001",   -- 1
             "1011011" when "0010",   -- 2
             "1001111" when "0011",   -- 3
             "1100110" when "0100",   -- 4
             "1101101" when "0101",   -- 5
             "1111101" when "0110",   -- 6
             "0000111" when "0111",   -- 7
             "1111111" when "1000",   -- 8
             "1101111" when "1001",   -- 9
             "1110111" when "1010",   -- A
             "1111100" when "1011",   -- b
             "0111001" when "1100",   -- C
             "1011110" when "1101",   -- d
             "1111001" when "1110",   -- E
             "1110001" when others;   -- F

   sseg(6 downto 0) <= not seg;
   sseg(7)          <= not dp;

end arch;
```

**A tabela.** Para cada valor de 0 a F, guarda quais dos 7 segmentos (a até g) precisam acender para desenhar aquele número ou letra na tela. É a mesma tabela de qualquer display de 7 segmentos por aí, só copiando o padrão visual conhecido. Não tem cálculo nenhum envolvido, é uma tabela de consulta direta.

**A inversão.** Na DE10-Lite os segmentos acendem com `0`, não com `1`: lógica ativa em nível baixo. Como é bem mais fácil escrever e revisar a tabela pensando em "`1` = aceso", ela foi montada do jeito normal e invertida de uma vez só no fim, com `not seg`. O ponto decimal (`dp`) segue a mesma regra, por isso o `not` também aparece na linha de baixo.

---

### 4.3 Funcionamento na Placa

Abaixo, imagens do funcionamento na placa para os 6 casos testados (superando os 4 casos mínimos, incluindo casos com carry-out e com deslocamento grande na normalização).

| Caso | Operação                 | O que demonstra                                                                                                         |
| ---- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| 1    | 192 + 192 = 384          | Soma com carry-out, exigindo deslocamento da mantissa para a direita e incremento do expoente na normalização.          |
| 2    | 192 − 128 = 64           | Subtração com deslocamento de 1 bit na normalização (expoente vai de 8 para 7).                                         |
| 3    | 2,0625 − 2 = 0,0625      | Resultado forçado a zero por _underflow_, sem empate de magnitudes.                                                     |
| 4    | 8 − 8 = 0                | Resultado nulo por empate exato de magnitudes, validando o tratamento do caso especial (com sinal negativo "fantasma"). |
| 5    | 16384 + 0,984375 ≈ 16384 | Operando pequeno completamente "engolido" no alinhamento, por diferença de expoente grande demais para o formato.       |
| 6    | 132 − 128 = 4            | Subtração com deslocamento grande na normalização (5 bits), o "muito deslocamento" pedido pela professora.              |

Para os casos a seguir, as conversões são montadas assim: sinal = 0 (positivo), expoente igual à primeira potência de 2 acima do número, e fração de 8 bits obtida por arredondar $N / 2^e \times 256$. As chaves SW mapeiam sinal, expoente e os cinco bits centrais da fração (bits 6 a 2, já que o primeiro é fixo em 1 e os dois últimos em 0). O somador executa ordenação, alinhamento, soma e normalização, e o resultado é exibido como sinal, expoente hexadecimal e dois dígitos hex da fração, com valor recuperado por $\text{fração} / 256 \times 2^{\text{expoente}}$. O método será esclarecido em cada caso isolado.

---

#### Caso 1: 192 + 192 = 384 (carry-out)

Este caso força um _carry-out_ na soma dos significandos, testando o primeiro segmento de decisão da normalização (incremento do expoente).

Os dois operandos são iguais: 192.

- Primeira potência de 2 acima de 192 → `2^8 = 256`, então expoente = 8 (binário `1000`).
- `192 / 256 = 0,75`; `× 256 = 192` → fração = 192 = binário `11000000`.

Como A = B = 192, as mesmas chaves servem para os dois operandos:

| Operando | Valor | `SW9` (sinal) | `SW8`        | `SW7`     | `SW6`     | `SW5`     | `SW4`        | `SW3`     | `SW2`     | `SW1`     | `SW0`     |
| -------- | ----- | ------------- | ------------ | --------- | --------- | --------- | ------------ | --------- | --------- | --------- | --------- |
| A = B    | 192   | 0 (baixo)     | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) |

**Roteiro para testar na placa:**

1. Deixe as chaves `SW8` e `SW4` para cima, todas as outras para baixo → monta 192.
2. Aperte e solte `KEY0` para guardar 192 como operando A.
3. Não mexa em mais nada: o operando B já está com o mesmo valor (192) nas chaves.
4. O resultado (384) aparece direto nos displays (circuito combinacional).

**Os 4 estágios:**

Com A = B = 192 (exp 8, frac `11000000`):

1. **Ordenação:** magnitudes iguais; a escolha de qual é "grande"/"pequeno" não afeta o resultado, já que os operandos são idênticos.
2. **Alinhamento:** `exp_diff = 8 − 8 = 0`, então nenhuma fração precisa deslizar.
3. **Soma:** sinais iguais → soma: `11000000` (192) `+ 11000000` (192) `= 110000000` (384 em 9 bits, com bit de carry `sum(8) = 1`).
4. **Normalização:** como houve carry-out, o circuito desloca a fração 1 bit à direita e incrementa o expoente: `expn = 8 + 1 = 9`; `fracn = sum(8 downto 1) = 11000000` (192).

Resultado: sinal `+`, expoente 9, fração 192 (`11000000` = `0xC0`). Em decimal: `192 / 256 × 2^9 = 0,75 × 512 = 384`.

| Display | Mostra                   | Neste exemplo      |
| ------- | ------------------------ | ------------------ |
| `HEX3`  | sinal                    | apagado (positivo) |
| `HEX2`  | expoente, em hexadecimal | `9`                |
| `HEX1`  | 4 bits altos da fração   | `C` (`1100`)       |
| `HEX0`  | 4 bits baixos da fração  | `0`                |

![Caso 1: 192+192, carry-out, testado na placa](../doc/img//caso1_192mais192_carryout.png)

---

#### Caso 2: 192 − 128 = 64 (deslocamento de 1 bit na normalização)

Este caso testa um deslocamento pequeno (1 bit) no 4º estágio, complementando o Caso 6 (deslocamento de 5 bits) com um exemplo mais simples do mesmo mecanismo.

**Para o 192** (mesmo cálculo do Caso 1):

- expoente = 8 (`1000`); fração = 192 (`11000000`).

**Para o 128** (entra negativo, para fazer `192 + (−128)`):

- Primeira potência de 2 acima de 128 → `2^8 = 256`, então expoente = 8 (`1000`).
- `128 / 256 = 0,5`; `× 256 = 128` → fração = 128 = `10000000`.

| Operando | Valor | `SW9` (sinal) | `SW8`        | `SW7`     | `SW6`     | `SW5`     | `SW4`        | `SW3`     | `SW2`     | `SW1`     | `SW0`     |
| -------- | ----- | ------------- | ------------ | --------- | --------- | --------- | ------------ | --------- | --------- | --------- | --------- |
| A        | +192  | 0 (baixo)     | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) |
| B        | −128  | **1 (cima)**  | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo)    | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) |

**Roteiro para testar na placa:**

1. Suba `SW8` e `SW4` (demais para baixo) → monta 192.
2. Aperte e solte `KEY0` para guardar 192 como operando A.
3. Suba `SW9` (sinal negativo) e abaixe `SW4`, mantendo `SW8` levantado → representa −128 como operando B.
4. O resultado (64) aparece direto nos displays.

**Os 4 estágios:**

Com A = 192 (exp 8, frac `11000000`, sinal +) e B = 128 (exp 8, frac `10000000`, sinal −):

1. **Ordenação:** mesmo expoente; compara frações: `192 > 128`, logo A é o "grande" (positivo) e B o "pequeno" (negativo).
2. **Alinhamento:** `exp_diff = 8 − 8 = 0`, nenhuma fração desliza.
3. **Subtração:** sinais diferentes → subtrai: `11000000` (192) `− 10000000` (128) `= 01000000` (64), sem carry (`sum(8) = 0`).
4. **Normalização:** contando zeros à esquerda em `01000000`, o primeiro `1` aparece logo no bit 6 → `leado = 1`. Como `leado (1)` não é maior que `expb (8)`, é o caso normal: desloca a fração 1 bit à esquerda e decrementa o expoente: `expn = 8 − 1 = 7`; `fracn = 10000000` (128).

Resultado: sinal `+`, expoente 7, fração 128 (`10000000` = `0x80`). Em decimal: `128 / 256 × 2^7 = 0,5 × 128 = 64`. E de fato 192 − 128 = 64.

| Display | Mostra                   | Neste exemplo      |
| ------- | ------------------------ | ------------------ |
| `HEX3`  | sinal                    | apagado (positivo) |
| `HEX2`  | expoente, em hexadecimal | `7`                |
| `HEX1`  | 4 bits altos da fração   | `8`                |
| `HEX0`  | 4 bits baixos da fração  | `0`                |

![Caso 2: 192−128, deslocamento de 1 bit, testado na placa](../doc/img//caso2_192menos128_deslocamento1bit.png)

---

#### Caso 3: 2,0625 − 2 = 0,0625 (underflow, sem empate de magnitudes)

Este caso mostra um segundo caminho para o resultado ser forçado a zero: não por empate exato de magnitudes (como no Caso 4), mas porque a diferença real é menor do que o menor número normalizado representável no formato (`0,5 × 2⁰ = 0,5`).

**Para o 2,0625:**

- Primeira potência de 2 acima de 2,0625 → `2^2 = 4`, então expoente = 2 (`0010`).
- `2,0625 / 4 = 0,515625`; `× 256 = 132` → fração = 132 = `10000100`.

**Para o 2** (entra negativo, para fazer `2,0625 + (−2)`):

- Primeira potência de 2 acima de 2 → `2^2 = 4` (não pode ser `2^1 = 2`, pois `2 / 2 = 1,0` fica fora da faixa `[0,5 , 1)`), então expoente = 2.
- `2 / 4 = 0,5`; `× 256 = 128` → fração = 128 = `10000000`.

| Operando | Valor   | `SW9` (sinal) | `SW8`     | `SW7`     | `SW6`        | `SW5`     | `SW4`     | `SW3`     | `SW2`     | `SW1`     | `SW0`        |
| -------- | ------- | ------------- | --------- | --------- | ------------ | --------- | --------- | --------- | --------- | --------- | ------------ |
| A        | +2,0625 | 0 (baixo)     | 0 (baixo) | 0 (baixo) | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | **1 (cima)** |
| B        | −2      | **1 (cima)**  | 0 (baixo) | 0 (baixo) | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo)    |

**Roteiro para testar na placa:**

1. Suba `SW6` e `SW0` (demais para baixo) → monta 2,0625.
2. Aperte e solte `KEY0` para guardar como operando A.
3. Suba `SW9` (sinal negativo), mantenha `SW6` levantado, abaixe `SW0` → representa −2 como operando B.
4. O resultado (zero) aparece direto nos displays.

**Os 4 estágios:**

Com A = 2,0625 (exp 2, frac `10000100`, sinal +) e B = 2 (exp 2, frac `10000000`, sinal −):

1. **Ordenação:** mesmo expoente; compara frações: `132 > 128`, logo A é o "grande" (positivo) e B o "pequeno" (negativo).
2. **Alinhamento:** `exp_diff = 0`, nenhuma fração desliza.
3. **Subtração:** sinais diferentes → subtrai: `10000100 − 10000000 = 00000100` (4), sem carry.
4. **Normalização:** contando zeros à esquerda em `00000100`: o primeiro `1` só aparece no bit 2, então `leado = 5`. Como `leado (5) > expb (2)`, o circuito reconhece que o deslocamento necessário é maior do que o próprio expoente disponível (o número é pequeno demais para ser normalizado) e força `expn = 0`, `fracn = 0`.

Resultado: expoente 0, fração 0 → zero. Como A (positivo) foi o "grande" desta vez, `sign_out = 0`, então o `HEX3` fica apagado, diferente do Caso 4, onde o "grande" era o operando negativo.

| Display | Mostra                   | Neste exemplo      |
| ------- | ------------------------ | ------------------ |
| `HEX3`  | sinal                    | apagado (positivo) |
| `HEX2`  | expoente, em hexadecimal | `0`                |
| `HEX1`  | 4 bits altos da fração   | `0`                |
| `HEX0`  | 4 bits baixos da fração  | `0`                |

![Caso 3: 2,0625−2, underflow sem empate, testado na placa](../doc/img//caso3_2p0625menos2_underflow.png)

---

#### Caso 4: 8 − 8 = 0 (resultado nulo)

Este caso valida a condição especial de zero: quando o resultado é pequeno demais para ser normalizado, o circuito força expoente e fração em zero. No entanto, como descobrimos durante a apresentação (será detalhado no Caso 7), alguns casos de resultado nulo mostram o expoente diferente de zero, embora a fração seja sempre nula.

**Para o 8:**

- Primeira potência de 2 acima de 8 → `2^4 = 16`, então expoente = 4 (`0100`).
- `8 / 16 = 0,5`; `× 256 = 128` → fração = 128 = `10000000`.

O segundo operando é o mesmo valor (8), mas com o sinal negativo, para fazer `8 + (−8)`.

| Operando | Valor | `SW9` (sinal) | `SW8`     | `SW7`        | `SW6`     | `SW5`     | `SW4`     | `SW3`     | `SW2`     | `SW1`     | `SW0`     |
| -------- | ----- | ------------- | --------- | ------------ | --------- | --------- | --------- | --------- | --------- | --------- | --------- |
| A        | +8    | 0 (baixo)     | 0 (baixo) | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) |
| B        | −8    | **1 (cima)**  | 0 (baixo) | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) |

**Roteiro para testar na placa:**

1. Monte +8 nas chaves (só `SW7` para cima) e aperte `KEY0` para guardar como operando A.
2. Suba também `SW9` (sinal negativo), mantendo `SW7` para cima → as chaves agora representam −8 (operando B).
3. O resultado (zero) aparece direto nos displays.

**Os 4 estágios:**

Com A = +8 e B = −8 (ambos exp 4, frac `10000000`):

1. **Ordenação:** como expoente e fração de A e B são idênticos, o circuito não encontra um "maior" estrito (`exp1&frac1 > exp2&frac2` é falso) e, pelo critério de desempate do VHDL, atribui B como número grande (`b`) e A como número pequeno (`s`); isso leva o sinal negativo de B para `signb`.
2. **Alinhamento:** `exp_diff = 0`, nenhuma fração desliza.
3. **Subtração:** sinais diferentes (`signb` negativo, `signs` positivo) → subtrai: `10000000 − 10000000 = 00000000`, sem carry.
4. **Normalização:** o resultado é todo zero, então nenhum bit de `sum(7 downto 1)` está em `'1'` e o contador de zeros cai no caso padrão (`leado = "111"`, que em binário dá 7). Como `leado (7) > expb (4)`, o circuito reconhece que o número é pequeno demais para ser normalizado e força `expn = 0` e `fracn = 0`.

Resultado: expoente 0, fração 0, ou seja, zero.

> **Por que o sinal "sobra" apesar do resultado ser zero?** Porque o `sign_out` é decidido logo no 1º estágio do `fp_adder`, antes de qualquer soma, apenas com uma comparação estrita (`>`) entre as magnitudes (`exp1&frac1` × `exp2&frac2`). Se os valores são exatamente iguais (como 8 e 8), ninguém é “maior”, o circuito cai no `else` e copia o sinal do segundo operando — que neste teste é negativo. Esse sinal fica gravado em `signb` e vai direto para `sign_out`, sem nunca ser reavaliado.
>
> O quarto estágio, bem mais tarde, detecta que o resultado é zero e zera `exp_out` e `frac_out`, mas já não consegue “avisar” o primeiro estágio para também zerar o sinal — ele já foi definido e propagado. Resultado: os dígitos mostram zero corretamente, mas o traço negativo pode continuar aparecendo no `HEX3`.

| Display   | Mostra                     | Neste exemplo                               |
| --------- | -------------------------- | ------------------------------------------- |
| `HEX3`    | sinal                      | pode mostrar o traço (ver observação acima) |
| `HEX2`    | expoente, em hexadecimal   | `0`                                         |
| `HEX1`    | 4 bits altos da fração     | `0`                                         |
| `HEX0`    | 4 bits baixos da fração    | `0`                                         |
| `LEDR(8)` | flag de zero (`zero_flag`) | aceso (`1`)                                 |

![Caso 4: 8−8, sinal negativo no zero, testado na placa](../doc/img//caso4_8menos8_zero_negativo.png)

---

#### Caso 5: 16384 + 0,984375 ≈ 16384 (operando pequeno "engolido")

Este caso mostra o que acontece quando a diferença de expoentes entre os dois operandos é grande demais. A fração do operando menor desliza para fora de todos os 8 bits durante o alinhamento e some por completo, como se fosse somado zero.

**Para o 16384:**

- Primeira potência de 2 acima → `2^15 = 32768`, então expoente = 15 (`1111`).
- `16384 / 32768 = 0,5`; `× 256 = 128` → fração = 128 = `10000000`.

**Para o 0,984375:**

- Primeira potência de 2 acima → `2^0 = 1`, então expoente = 0 (`0000`).
- `0,984375 / 1 = 0,984375`; `× 256 = 252` → fração = 252 = `11111100`.

| Operando | Valor     | `SW9` (sinal) | `SW8`        | `SW7`        | `SW6`        | `SW5`        | `SW4`        | `SW3`        | `SW2`        | `SW1`        | `SW0`        |
| -------- | --------- | ------------- | ------------ | ------------ | ------------ | ------------ | ------------ | ------------ | ------------ | ------------ | ------------ |
| A        | +16384    | 0 (baixo)     | **1 (cima)** | **1 (cima)** | **1 (cima)** | **1 (cima)** | 0 (baixo)    | 0 (baixo)    | 0 (baixo)    | 0 (baixo)    | 0 (baixo)    |
| B        | +0,984375 | 0 (baixo)     | 0 (baixo)    | 0 (baixo)    | 0 (baixo)    | 0 (baixo)    | **1 (cima)** | **1 (cima)** | **1 (cima)** | **1 (cima)** | **1 (cima)** |

**Roteiro para testar na placa:**

1. Suba `SW8`, `SW7`, `SW6` e `SW5` (demais para baixo) → monta 16384.
2. Aperte e solte `KEY0` para guardar como operando A.
3. Abaixe `SW8` a `SW5` e suba todas as chaves da fração (`SW4` a `SW0`) → representa 0,984375 como operando B.
4. O resultado aparece direto nos displays.

**Os 4 estágios:**

Com A = 16384 (exp 15, frac `10000000`) e B = 0,984375 (exp 0, frac `11111100`):

1. **Ordenação:** A tem expoente muitíssimo maior, então vira o "número grande"; B é o "pequeno".
2. **Alinhamento:** `exp_diff = 15 − 0 = 15`. O deslocamento necessário é maior do que os 8 bits da fração de B: ela desliza inteira para fora e `fraca` vira `00000000`. O operando pequeno é completamente descartado nesta etapa, antes mesmo da soma.
3. **Soma:** sinais iguais → soma: `10000000 + 00000000 = 10000000` (128), sem carry.
4. **Normalização:** a fração já começa com `1` → nenhum deslocamento. `expn = expb = 15`.

Resultado: sinal `+`, expoente 15, fração 128 (`0x80`), exatamente igual ao valor original de A. Somar 0,984 a 16384 não mudou nada visível no resultado, porque a diferença de magnitude é grande demais para o formato de 13 bits capturar.

| Display | Mostra                   | Neste exemplo      |
| ------- | ------------------------ | ------------------ |
| `HEX3`  | sinal                    | apagado (positivo) |
| `HEX2`  | expoente, em hexadecimal | `F`                |
| `HEX1`  | 4 bits altos da fração   | `8`                |
| `HEX0`  | 4 bits baixos da fração  | `0`                |

![Caso 5: 16384+0,984, operando engolido, testado na placa](../doc/img//caso5_16384mais0p984_engolido.png)

---

#### Caso 6: 132 − 128 = 4 (deslocamento de 5 bits na normalização)

Este caso testa o segmento de contagem/deslocamento de zeros à esquerda no 4º estágio, usado quando a subtração produz um resultado com vários bits mais significativos zerados. Escolhemos um par de números com deslocamento maior (5 bits) para deixar bem evidente o grande deslocamento nesta etapa.

**Para o 132:**

- Primeira potência de 2 acima de 132 → `2^8 = 256`, então expoente = 8 (`1000`).
- `132 / 256 = 0,515625`; `× 256 = 132` → fração = 132 = `10000100`.

**Para o 128:**

- Primeira potência de 2 acima de 128 → `2^8 = 256`, então expoente = 8.
- `128 / 256 = 0,5`; `× 256 = 128` → fração = 128 = `10000000`.

Como é uma subtração (132 − 128), o 128 entra com o bit de sinal ligado (negativo): o somador soma quando os sinais são iguais e subtrai quando são diferentes, então isso equivale a calcular `132 + (−128)`.

| Operando | Valor | `SW9` (sinal) | `SW8`        | `SW7`     | `SW6`     | `SW5`     | `SW4`     | `SW3`     | `SW2`     | `SW1`     | `SW0`        |
| -------- | ----- | ------------- | ------------ | --------- | --------- | --------- | --------- | --------- | --------- | --------- | ------------ |
| A        | +132  | 0 (baixo)     | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | **1 (cima)** |
| B        | −128  | **1 (cima)**  | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo)    |

**Roteiro para testar na placa:**

1. Suba `SW8` e `SW0` (demais para baixo) → monta 132.
2. Aperte e solte `KEY0` para guardar como operando A.
3. Suba também `SW9` (sinal negativo), mantenha `SW8` levantado e abaixe `SW0` → representa −128 como operando B.
4. O resultado (4) aparece direto nos displays.

**Os 4 estágios:**

Com A = 132 (exp 8, frac `10000100`, sinal +) e B = 128 (exp 8, frac `10000000`, sinal −):

1. **Ordenação:** mesmo expoente, então o circuito compara as frações: `132 > 128`, logo A (132) vira o "número grande" (`b`) e B (128) o "número pequeno" (`s`).
2. **Alinhamento:** `exp_diff = 8 − 8 = 0`, nenhuma fração desliza.
3. **Subtração:** sinais diferentes → subtrai: `10000100` (132) `− 10000000` (128) `= 00000100` (4), sem carry (`sum(8) = 0`).
4. **Normalização:** contando os zeros à esquerda em `00000100`, o primeiro `1` só aparece no bit 2 → `leado = 5`. Como `leado (5)` não é maior que `expb (8)`, é o caso normal: o circuito desloca a fração 5 bits à esquerda e decrementa o expoente: `expn = 8 − 5 = 3`; `fracn = 10000000` (128).

Resultado: sinal `+` (herdado do operando de maior magnitude, 132), expoente 3, fração 128 (`10000000` = `0x80`). Em decimal: `128 / 256 × 2^3 = 0,5 × 8 = 4`.

| Display | Mostra                   | Neste exemplo      |
| ------- | ------------------------ | ------------------ |
| `HEX3`  | sinal                    | apagado (positivo) |
| `HEX2`  | expoente, em hexadecimal | `3`                |
| `HEX1`  | 4 bits altos da fração   | `8`                |
| `HEX0`  | 4 bits baixos da fração  | `0`                |

![Caso 6: 132−128, deslocamento de 5 bits, testado na placa](../doc/img//caso6_132menos128_muitodeslocamento.png)

---

#### Caso 7 (sem foto): 992 − 992 = 0, mas o expoente não zera

Descobrimos este caso durante a apresentação do trabalho, por isso não registramos foto. Nele, como no Caso 4, um número é subtraído de si mesmo e, assim, a fração é zerada. Contudo, há casos em que o expoente continua diferente de zero, como em 992 - 992.

**Para o 992:**

- A primeira potência de 2 acima dele é `2^10 = 1024`, então o expoente é 10 (`1010`). `992 / 1024 = 0,96875`, e multiplicando por 256 dá `248`, então a fração é 248 (`11111000`).

O segundo operando é o mesmo valor, só que negativo, pra fazer `992 + (−992)`.

| Operando | Valor | `SW9` (sinal) | `SW8`        | `SW7`     | `SW6`        | `SW5`     | `SW4`        | `SW3`        | `SW2`        | `SW1`        | `SW0`     |
| -------- | ----- | ------------- | ------------ | --------- | ------------ | --------- | ------------ | ------------ | ------------ | ------------ | --------- |
| A        | +992  | 0 (baixo)     | **1 (cima)** | 0 (baixo) | **1 (cima)** | 0 (baixo) | **1 (cima)** | **1 (cima)** | **1 (cima)** | **1 (cima)** | 0 (baixo) |
| B        | −992  | **1 (cima)**  | **1 (cima)** | 0 (baixo) | **1 (cima)** | 0 (baixo) | **1 (cima)** | **1 (cima)** | **1 (cima)** | **1 (cima)** | 0 (baixo) |

**Roteiro para testar na placa**

1. Suba `SW8` e `SW6` (expoente 10), demais chaves de expoente para baixo.
2. Suba `SW4`, `SW3`, `SW2` e `SW1` (fração 248), deixe `SW0` para baixo.
3. Aperte e solte `KEY0` para guardar 992 como operando A.
4. Suba também `SW9` (sinal negativo), mantendo o resto igual. Isso representa −992 como operando B.
5. O resultado aparece direto nos displays.

**Os 4 estágios:**

Com A = +992 (exp 10, frac `11111000`) e B = −992 (exp 10, frac `11111000`):

1. **Ordenação:** magnitude idêntica nos dois operandos, então a comparação `exp1&frac1 > exp2&frac2` dá falso, e o VHDL cai no `else`: B vira o número grande (`b`) e A o número pequeno (`s`). Isso leva o sinal negativo de B para `signb`, o mesmo mecanismo do Caso 4.
2. **Alinhamento:** `exp_diff = 10 − 10 = 0`, nenhuma fração desliza.
3. **Subtração:** sinais diferentes, então `11111000 − 11111000 = 00000000`, sem carry.
4. **Normalização:** o resultado é todo zero, então nenhum bit de `sum(7 downto 1)` está em `'1'`, e o contador de zeros cai no valor padrão do `case`: `leado = "111"` (7 em decimal). O sinal `leado` só tem 3 bits, então esse é o valor máximo que ele consegue representar, mesmo quando o resultado tem, na prática, 8 posições zeradas. O teste seguinte do código é `leado > expb`. Com `expb = 10`, a comparação `7 > 10` é falsa, então o circuito não entra no ramo que zera o resultado, e cai no ramo normal: `expn = expb − leado = 10 − 7 = 3`; `fracn = sum_norm`, que para `leado = "111"` é `sum(0) & "0000000"`, ou seja `00000000`.

Resultado: sinal negativo (herdado de B), expoente 3 (`0011`), fração 0. A fração e o sinal saem certos, mas o expoente sai 3 em vez de 0.

| Display | Mostra                   | Neste exemplo                        |
| ------- | ------------------------ | ------------------------------------ |
| `HEX3`  | sinal                    | traço aceso (negativo, herdado de B) |
| `HEX2`  | expoente, em hexadecimal | `3` (errado, devia ser `0`)          |
| `HEX1`  | 4 bits altos da fração   | `0`                                  |
| `HEX0`  | 4 bits baixos da fração  | `0`                                  |

Por que isso não acontece no Caso 4 (`8 − 8`)? Lá o expoente do 8 é 4. A mesma conta dá `leado = 7`, e o teste `leado (7) > expb (4)` é verdadeiro, então o circuito cai no ramo que força tudo a zero. O ponto de virada é o expoente 7: `leado (7) > expb (7)` já é falso, então o circuito cai no ramo normal e calcula `expn = 7 − 7 = 0`, que por coincidência ainda dá certo. O erro só fica visível a partir do expoente 8, quando `expn = expb − 7` deixa de dar zero.

A causa é o próprio sinal `leado`, declarado no VHDL com só 3 bits (`signal leado : unsigned(2 downto 0)`). Três bits alcançam no máximo o valor 7, mas para sinalizar "nenhum bit 1 encontrado nos 8 bits da soma" seria preciso o valor 8, que não cabe nesse tamanho. O teste `leado > expb` só detecta esse underflow por coincidência, enquanto o expoente for pequeno o bastante. Uma pequena limitação do algoritmo original do livro.
