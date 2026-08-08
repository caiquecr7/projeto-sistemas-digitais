## 4. Evidências de Validação

### Simulação

A imagem abaixo é do Questa (ModelSim), rodando o testbench `fp_adder_de10lite_vhd_tst` em cima do top-level já com os pinos da placa. Não é só um dump de sinais: montamos o testbench para ser **autoverificável**: ele aplica uma sequência de combinações de switches e pulsos de `KEY`, compara o resultado que sai do `fp_adder` com o valor esperado calculado à parte, e vai contando num sinal chamado `erros` toda vez que gera diferença. No fim da simulação, o sinal `fim_sim` sobe para `1`.

No print, o cursor está em 1440 ns, perto do fim da simulação. Nesse instante os dois sinais de controle do testbench mostram exatamente o que a gente queria ver: `fim_sim = TRUE` e `erros = 0`. Rodou a bateria inteira de vetores de teste e nenhum resultado do `fp_adder` divergiu do valor esperado calculado à parte no testbench. Essa é uma verificação mais confiável do que apenas olhar a placa: aqui é o próprio testbench comparando automaticamente cada resultado, não uma pessoa conferindo à mão caso por caso.

Podemos confirmar o mecanismo olhando a trilha do `KEY`: ela fica alternando entre o valor `3` (`"11"`, os dois botões soltos, o estado parado entre um vetor e outro) e o valor `2` (`"10"`, ou seja `KEY(0) = '0'` pressionado) repetidas vezes ao longo da simulação, que é justamente o `KEY0` carregando um novo operando A antes de cada novo teste. Perto do fim aparece também um único pulso com o valor `1` (`"01"`, `KEY(1) = '0'`): o `KEY1` limpando o operando A, provavelmente um vetor à parte só pra conferir que o clear funciona. Contando essas trocas junto com as mudanças na trilha de `SW`, dá pra ver mais de dez configurações de entrada diferentes passando ao longo da simulação, cada uma seguida do pulso de `KEY` correspondente, o mesmo mecanismo que a gente testou na mão na Etapa 3, só que aqui automatizado.

![Simulação no Questa: testbench autoverificável, erros = 0 ao final da execução](../doc/img/simulacao_testbench_autocheck.png)

### Código VHDL Final

#### 1. Top-level adaptado para a DE10-Lite (`fp_adder_de10lite.vhd`)

Esse é o módulo de topo: o que realmente vira bitstream e roda na placa. Ele substitui o `fp_adder_test` do livro (Listing 3.20), pensado para uma placa antiga com 8 chaves, 4 botões e display multiplexado, que não é compatível com a DE10-Lite. Por isso quase tudo aqui foi reescrito; a única peça que entra sem alteração é o próprio `fp_adder`, chamado como componente.

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

**Portas.** Trocamos os nomes genéricos do livro pelos nomes reais dos pinos da placa: `SW`, `KEY`, `HEX0` a `HEX5` e `LEDR` já são reconhecidos automaticamente pelo Quartus a partir do `.qsf` da DE10-Lite. Também entrou o `MAX10_CLK1_50`, que não existia na versão original: precisamos dele porque agora tem um elemento sequencial no circuito.

**Operando B.** Não tem segredo, é ligação direta das chaves: `SW(9)` vira o sinal, `SW(8 downto 5)` vira o expoente. A fração é montada concatenando um `1` fixo (para garantir que o número entra normalizado), os 5 switches que sobraram, e dois `0` no final. Restam só 5 dos 8 bits realmente controláveis, mas é o que dá para fazer com 10 chaves.

**Operando A e o registrador.** Essa é a mudança mais importante da etapa. Na primeira versão, A era uma constante fixa; só que com 10 switches não dá para controlar os 26 bits dos dois operandos ao mesmo tempo. A solução foi transformar A num registrador: aperta `KEY0` e ele copia o que está nas chaves naquele instante; aperta `KEY1` e ele zera. É o único trecho sequencial do projeto inteiro: todo o resto do circuito é combinacional puro e reage na hora, sem depender do clock.

**A instância do `fp_adder`.** Aqui é só um `port map`, ligando `signA/expA/fracA` e `signB/expB/fracB` nas entradas e recolhendo `sign_out/exp_out/frac_out` na saída. Não tem lógica nova nessa parte: é literalmente o componente da Etapa 1 sendo chamado sem mudanças.

**LEDs de depuração.** `LEDR(9)` repete o sinal do resultado, `LEDR(8)` acende quando a fração resulta em zero (útil para não precisar ficar decorando que `0x00` é zero), e os outros 8 mostram a fração crua em binário, bit a bit, como ela sai do somador.

**Displays.** Cada instância de `hex_to_sseg_de10` recebe 4 bits e devolve o padrão de segmentos correspondente. `HEX2` fica com o expoente, `HEX1` e `HEX0` dividem a fração nos dois nibbles. `HEX3` é tratado à parte porque não precisa decodificar nada em hexadecimal: só acende o segmento do meio quando o resultado é negativo, e fica todo apagado quando é positivo. `HEX4` e `HEX5` sobraram sem uso depois que a professora pediu para tirar a memória do operando A da tela, então ficam apagados o tempo todo.

#### 2. Decodificador de 7 segmentos adaptado (`hex_to_sseg_de10.vhd`)

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

> **Observação:** o arquivo `fp_adder.vhd` permaneceu exatamente como no original, apenas com correções de aspas e operadores que o PDF do livro havia distorcido. Nenhuma lógica matemática foi alterada.

---

### Funcionamento na Placa

Abaixo, imagens do funcionamento na placa para os 6 casos testados (superando os 4 casos mínimos, incluindo casos com carry-out e com deslocamento grande na normalização).

| Caso | Operação                 | O que demonstra                                                                                                         |
| ---- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| 1    | 192 + 192 = 384          | Soma com carry-out, exigindo deslocamento da mantissa para a direita e incremento do expoente na normalização.          |
| 2    | 192 − 128 = 64           | Subtração com deslocamento de 1 bit na normalização (expoente vai de 8 para 7).                                         |
| 3    | 2,0625 − 2 = 0,0625      | Resultado forçado a zero por _underflow_, sem empate de magnitudes.                                                     |
| 4    | 8 − 8 = 0                | Resultado nulo por empate exato de magnitudes, validando o tratamento do caso especial (com sinal negativo "fantasma"). |
| 5    | 16384 + 0,984375 ≈ 16384 | Operando pequeno completamente "engolido" no alinhamento, por diferença de expoente grande demais para o formato.       |
| 6    | 132 − 128 = 4            | Subtração com deslocamento grande na normalização (5 bits), o "muito deslocamento" pedido pela professora.              |

#### Exemplo de referência: 128 + 24 = 152 (ilustra o método; não foi fotografado na bancada)

Antes dos 6 casos testados na placa, vale mostrar como qualquer número é convertido para o formato de 13 bits, usando um exemplo simples: **128 + 24 = 152**. Escolhemos esses números porque encaixam exatamente nas chaves da placa e, além disso, o 24 mostra como usar os bits do meio da fração (`SW4` a `SW0`).

##### Escrevendo um número no formato de 13 bits:

Para colocar um número `N` (positivo) no formato, são três passos:

1. **Sinal:** positivo → `0`.
2. **Expoente:** pegue a primeira potência de 2 acima de `N`. Se ela é `2^e`, o campo de expoente é `e`.
3. **Fração:** calcule `frac = arredonda( N / 2^e × 256 )`, que dá os 8 bits do significando.

**Para o 128:**

- Primeira potência de 2 acima de 128 → `2^8 = 256`, então expoente = 8 (`1000`).
- `128 / 256 = 0,5`; `× 256 = 128` → fração = 128 = `10000000`.

**Para o 24:**

- Primeira potência de 2 acima de 24 → `2^5 = 32`, então expoente = 5 (`0101`).
- `24 / 32 = 0,75`; `× 256 = 192` → fração = 192 = `11000000`.

##### Quais chaves `SW` acionar:

As 10 chaves são divididas assim: `SW9` é o sinal, `SW8` a `SW5` são o expoente (4 bits) e `SW4` a `SW0` são os 5 bits do meio da fração. O primeiro bit da fração é sempre fixo em `1` e os dois últimos bits são sempre fixos em `0`, então os 5 bits do meio das chaves correspondem aos bits `frac(6..2)` calculados acima.

- Para o 128, a fração é `10000000`. Os bits do meio (`frac(6..2)`) são `00000`, ou seja, nenhuma chave da fração precisa subir.
- Para o 24, a fração é `11000000`. Os bits do meio (`frac(6..2)`) são `10000`, ou seja, a primeira chave da fração (`SW4`) precisa subir, e as demais (`SW3` a `SW0`) ficam para baixo.

| Operando | Valor | `SW9` (sinal) | `SW8`        | `SW7`        | `SW6`     | `SW5`        | `SW4`        | `SW3`     | `SW2`     | `SW1`     | `SW0`     |
| -------- | ----- | ------------- | ------------ | ------------ | --------- | ------------ | ------------ | --------- | --------- | --------- | --------- |
| A        | 128   | 0 (baixo)     | **1 (cima)** | 0 (baixo)    | 0 (baixo) | 0 (baixo)    | 0 (baixo)    | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) |
| B        | 24    | 0 (baixo)     | 0 (baixo)    | **1 (cima)** | 0 (baixo) | **1 (cima)** | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) |

##### Roteiro para testar na placa:

1. Deixe só a chave **`SW8`** para cima (todas as outras para baixo) → isso monta o número 128.
2. Aperte e solte **`KEY0`** para guardar esse valor como operando A.
3. Agora deixe as chaves **`SW7`**, **`SW5`** e **`SW4`** para cima, e todas as outras para baixo → isso monta o número 24 como operando B.
4. O resultado aparece na hora nos displays (o circuito é combinacional, não precisa apertar nada para ver a soma).

##### A conta passo a passo (os 4 estágios):

Com A = 128 (exp 8, frac `10000000`) e B = 24 (exp 5, frac `11000000`):

1. **Ordenação:** o circuito vê que 128 (expoente 8) é o maior e 24 (expoente 5) é o menor.
2. **Alinhamento:** a diferença de expoente é `8 − 5 = 3`, então a fração de 24 (`11000000`) desliza 3 casas para a direita e vira `00011000` (= 24).
3. **Soma:** os sinais são iguais, então soma: `10000000` (128) `+ 00011000` (24) `= 010011000` (152). Não passou de 8 bits, então não teve "vai um".
4. **Normalização:** a fração já começa com `1`, então não precisa deslocar nada. O expoente continua em 8.

Resultado: sinal `+`, expoente 8, fração 152 (`10011000` = `0x98`). Em decimal: `152 / 256 × 2^8 = 152`. E de fato 128 + 24 = 152.

##### Lendo o resultado nos displays:

Com o mapeamento pedido pela professora (mostrando só o resultado):

| Display | Mostra                    | Neste exemplo      |
| ------- | ------------------------- | ------------------ |
| `HEX3`  | sinal (traço se negativo) | apagado (positivo) |
| `HEX2`  | expoente, em hexadecimal  | `8` (= 8)          |
| `HEX1`  | 4 bits altos da fração    | `9`                |
| `HEX0`  | 4 bits baixos da fração   | `8`                |

Para voltar ao decimal, é só aplicar a fórmula do formato:

$$\text{valor} = \frac{\text{fração}}{256} \times 2^{\text{expoente}} = \frac{\text{0x98}}{256} \times 2^{\text{0x8}} = \frac{152}{256} \times 256 = 152$$

---

#### Caso 1: 192 + 192 = 384 (carry-out)

Este caso força um _carry-out_ na soma dos significandos, testando o primeiro segmento de decisão da normalização (incremento do expoente).

##### Escrevendo os números no formato de 13 bits:

Os dois operandos são iguais: 192.

- Primeira potência de 2 acima de 192 → `2^8 = 256`, então **expoente = 8** (`1000`).
- `192 / 256 = 0,75`; `× 256 = 192` → fração = 192 = `11000000`.

##### Quais chaves `SW` acionar:

Como A = B = 192, as mesmas chaves servem para os dois operandos:

| Operando | Valor | `SW9` (sinal) | `SW8`        | `SW7`     | `SW6`     | `SW5`     | `SW4`        | `SW3`     | `SW2`     | `SW1`     | `SW0`     |
| -------- | ----- | ------------- | ------------ | --------- | --------- | --------- | ------------ | --------- | --------- | --------- | --------- |
| A = B    | 192   | 0 (baixo)     | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) |

##### Roteiro para testar na placa:

1. Deixe as chaves `SW8` e `SW4` para cima, todas as outras para baixo → monta 192.
2. Aperte e solte `KEY0` para guardar 192 como operando A.
3. Não mexa em mais nada: o operando B já está com o mesmo valor (192) nas chaves.
4. O resultado (384) aparece direto nos displays (circuito combinacional).

##### A conta passo a passo (os 4 estágios):

Com A = B = 192 (exp 8, frac `11000000`):

1. **Ordenação:** magnitudes iguais; a escolha de qual é "grande"/"pequeno" não afeta o resultado, já que os operandos são idênticos.
2. **Alinhamento:** `exp_diff = 8 − 8 = 0`, então nenhuma fração precisa deslizar.
3. **Soma:** sinais iguais → soma: `11000000` (192) `+ 11000000` (192) `= 110000000` (384 em 9 bits, com bit de carry `sum(8) = 1`).
4. **Normalização:** como houve carry-out, o circuito desloca a fração 1 bit à direita e incrementa o expoente: `expn = 8 + 1 = 9`; `fracn = sum(8 downto 1) = 11000000` (192).

Resultado: sinal `+`, expoente 9, fração 192 (`11000000` = `0xC0`). Em decimal: `192 / 256 × 2^9 = 0,75 × 512 = 384`.

##### Lendo o resultado nos displays:

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

##### Escrevendo os números no formato de 13 bits:

**Para o 192** (mesmo cálculo do Caso 1):

- expoente = 8 (`1000`); fração = 192 (`11000000`).

**Para o 128** (entra negativo, para fazer `192 + (−128)`):

- Primeira potência de 2 acima de 128 → `2^8 = 256`, então expoente = 8 (`1000`).
- `128 / 256 = 0,5`; `× 256 = 128` → fração = 128 = `10000000`.

##### Quais chaves `SW` acionar:

| Operando | Valor | `SW9` (sinal) | `SW8`        | `SW7`     | `SW6`     | `SW5`     | `SW4`        | `SW3`     | `SW2`     | `SW1`     | `SW0`     |
| -------- | ----- | ------------- | ------------ | --------- | --------- | --------- | ------------ | --------- | --------- | --------- | --------- |
| A        | +192  | 0 (baixo)     | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) |
| B        | −128  | **1 (cima)**  | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo)    | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) |

##### Roteiro para testar na placa:

1. Suba `SW8` e `SW4` (demais para baixo) → monta 192.
2. Aperte e solte `KEY0` para guardar 192 como operando A.
3. Suba `SW9` (sinal negativo) e abaixe `SW4`, mantendo `SW8` levantado → representa −128 como operando B.
4. O resultado (64) aparece direto nos displays.

##### A conta passo a passo (os 4 estágios):

Com A = 192 (exp 8, frac `11000000`, sinal +) e B = 128 (exp 8, frac `10000000`, sinal −):

1. **Ordenação:** mesmo expoente; compara frações: `192 > 128`, logo A é o "grande" (positivo) e B o "pequeno" (negativo).
2. **Alinhamento:** `exp_diff = 8 − 8 = 0`, nenhuma fração desliza.
3. **Subtração:** sinais diferentes → subtrai: `11000000` (192) `− 10000000` (128) `= 01000000` (64), sem carry (`sum(8) = 0`).
4. **Normalização:** contando zeros à esquerda em `01000000`, o primeiro `1` aparece logo no bit 6 → `leado = 1`. Como `leado (1)` não é maior que `expb (8)`, é o caso normal: desloca a fração 1 bit à esquerda e decrementa o expoente: `expn = 8 − 1 = 7`; `fracn = 10000000` (128).

Resultado: sinal `+`, expoente 7, fração 128 (`10000000` = `0x80`). Em decimal: `128 / 256 × 2^7 = 0,5 × 128 = 64`. E de fato 192 − 128 = 64.

##### Lendo o resultado nos displays:

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

##### Escrevendo os números no formato de 13 bits:

**Para o 2,0625:**

- Primeira potência de 2 acima de 2,0625 → `2^2 = 4`, então expoente = 2 (`0010`).
- `2,0625 / 4 = 0,515625`; `× 256 = 132` → fração = 132 = `10000100`.

**Para o 2** (entra negativo, para fazer `2,0625 + (−2)`):

- Primeira potência de 2 acima de 2 → `2^2 = 4` (não pode ser `2^1 = 2`, pois `2 / 2 = 1,0` fica fora da faixa `[0,5 , 1)`), então expoente = 2.
- `2 / 4 = 0,5`; `× 256 = 128` → fração = 128 = `10000000`.

##### Quais chaves `SW` acionar:

| Operando | Valor   | `SW9` (sinal) | `SW8`     | `SW7`     | `SW6`        | `SW5`     | `SW4`     | `SW3`     | `SW2`     | `SW1`     | `SW0`        |
| -------- | ------- | ------------- | --------- | --------- | ------------ | --------- | --------- | --------- | --------- | --------- | ------------ |
| A        | +2,0625 | 0 (baixo)     | 0 (baixo) | 0 (baixo) | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | **1 (cima)** |
| B        | −2      | **1 (cima)**  | 0 (baixo) | 0 (baixo) | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo)    |

##### Roteiro para testar na placa:

1. Suba `SW6` e `SW0` (demais para baixo) → monta 2,0625.
2. Aperte e solte `KEY0` para guardar como operando A.
3. Suba `SW9` (sinal negativo), mantenha `SW6` levantado, abaixe `SW0` → representa −2 como operando B.
4. O resultado (zero) aparece direto nos displays.

##### A conta passo a passo (os 4 estágios):

Com A = 2,0625 (exp 2, frac `10000100`, sinal +) e B = 2 (exp 2, frac `10000000`, sinal −):

1. **Ordenação:** mesmo expoente; compara frações: `132 > 128`, logo A é o "grande" (positivo) e B o "pequeno" (negativo).
2. **Alinhamento:** `exp_diff = 0`, nenhuma fração desliza.
3. **Subtração:** sinais diferentes → subtrai: `10000100 − 10000000 = 00000100` (4), sem carry.
4. **Normalização:** contando zeros à esquerda em `00000100`: o primeiro `1` só aparece no bit 2, então `leado = 5`. Como `leado (5) > expb (2)`, o circuito reconhece que o deslocamento necessário é maior do que o próprio expoente disponível (o número é pequeno demais para ser normalizado) e força `expn = 0`, `fracn = 0`.

Resultado: expoente 0, fração 0 → zero. Como A (positivo) foi o "grande" desta vez, `sign_out = 0`, então o `HEX3` fica apagado, diferente do Caso 4, onde o "grande" era o operando negativo.

##### Lendo o resultado nos displays:

| Display | Mostra                   | Neste exemplo      |
| ------- | ------------------------ | ------------------ |
| `HEX3`  | sinal                    | apagado (positivo) |
| `HEX2`  | expoente, em hexadecimal | `0`                |
| `HEX1`  | 4 bits altos da fração   | `0`                |
| `HEX0`  | 4 bits baixos da fração  | `0`                |

![Caso 3: 2,0625−2, underflow sem empate, testado na placa](../doc/img//caso3_2p0625menos2_underflow.png)

---

#### Caso 4: 8 − 8 = 0 (resultado nulo)

Este caso valida a condição especial de zero: quando o resultado é pequeno demais para ser normalizado, o circuito força expoente e fração em zero.

##### Escrevendo os números no formato de 13 bits:

**Para o 8:**

- Primeira potência de 2 acima de 8 → `2^4 = 16`, então expoente = 4 (`0100`).
- `8 / 16 = 0,5`; `× 256 = 128` → fração = 128 = `10000000`.

O segundo operando é o mesmo valor (8), mas com o sinal negativo, para fazer `8 + (−8)`.

##### Quais chaves `SW` acionar:

| Operando | Valor | `SW9` (sinal) | `SW8`     | `SW7`        | `SW6`     | `SW5`     | `SW4`     | `SW3`     | `SW2`     | `SW1`     | `SW0`     |
| -------- | ----- | ------------- | --------- | ------------ | --------- | --------- | --------- | --------- | --------- | --------- | --------- |
| A        | +8    | 0 (baixo)     | 0 (baixo) | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) |
| B        | −8    | **1 (cima)**  | 0 (baixo) | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) |

##### Roteiro para testar na placa:

1. Monte +8 nas chaves (só `SW7` para cima) e aperte `KEY0` para guardar como operando A.
2. Suba também `SW9` (sinal negativo), mantendo `SW7` para cima → as chaves agora representam −8 (operando B).
3. O resultado (zero) aparece direto nos displays.

##### A conta passo a passo (os 4 estágios):

Com A = +8 e B = −8 (ambos exp 4, frac `10000000`):

1. **Ordenação:** como expoente e fração de A e B são idênticos, o circuito não encontra um "maior" estrito (`exp1&frac1 > exp2&frac2` é falso) e, pelo critério de desempate do VHDL, atribui B como número grande (`b`) e A como número pequeno (`s`); isso leva o sinal negativo de B para `signb`.
2. **Alinhamento:** `exp_diff = 0`, nenhuma fração desliza.
3. **Subtração:** sinais diferentes (`signb` negativo, `signs` positivo) → subtrai: `10000000 − 10000000 = 00000000`, sem carry.
4. **Normalização:** o resultado é todo zero, então nenhum bit de `sum(7 downto 1)` está em `'1'` e o contador de zeros cai no caso padrão (`leado = "111"`, que em binário dá 7). Como `leado (7) > expb (4)`, o circuito reconhece que o número é pequeno demais para ser normalizado e força `expn = 0` e `fracn = 0`.

Resultado: expoente 0, fração 0, ou seja, zero.

> **Por que o sinal "sobra" apesar do resultado ser zero?** Porque o `sign_out` é decidido logo no 1º estágio do `fp_adder`, antes de qualquer soma, apenas com uma comparação estrita (`>`) entre as magnitudes (`exp1&frac1` × `exp2&frac2`). Se os valores são exatamente iguais (como 8 e 8), ninguém é “maior”, o circuito cai no `else` e copia o sinal do segundo operando — que neste teste é negativo. Esse sinal fica gravado em `signb` e vai direto para `sign_out`, sem nunca ser reavaliado.
>
> O quarto estágio, bem mais tarde, detecta que o resultado é zero e zera `exp_out` e `frac_out`, mas já não consegue “avisar” o primeiro estágio para também zerar o sinal — ele já foi definido e propagado. Resultado: os dígitos mostram zero corretamente, mas o traço negativo pode continuar aparecendo no `HEX3`.

##### Lendo o resultado nos displays:

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

Este caso mostra o que acontece quando a diferença de expoentes entre os dois operandos é grande demais: a fração do operando menor desliza para fora de todos os 8 bits durante o alinhamento e some por completo, como se fosse somado zero.

##### Escrevendo os números no formato de 13 bits:

**Para o 16384:**

- Primeira potência de 2 acima → `2^15 = 32768`, então expoente = 15 (`1111`).
- `16384 / 32768 = 0,5`; `× 256 = 128` → fração = 128 = `10000000`.

**Para o 0,984375:**

- Primeira potência de 2 acima → `2^0 = 1`, então expoente = 0 (`0000`).
- `0,984375 / 1 = 0,984375`; `× 256 = 252` → fração = 252 = `11111100`.

##### Quais chaves `SW` acionar:

| Operando | Valor     | `SW9` (sinal) | `SW8`        | `SW7`        | `SW6`        | `SW5`        | `SW4`        | `SW3`        | `SW2`        | `SW1`        | `SW0`        |
| -------- | --------- | ------------- | ------------ | ------------ | ------------ | ------------ | ------------ | ------------ | ------------ | ------------ | ------------ |
| A        | +16384    | 0 (baixo)     | **1 (cima)** | **1 (cima)** | **1 (cima)** | **1 (cima)** | 0 (baixo)    | 0 (baixo)    | 0 (baixo)    | 0 (baixo)    | 0 (baixo)    |
| B        | +0,984375 | 0 (baixo)     | 0 (baixo)    | 0 (baixo)    | 0 (baixo)    | 0 (baixo)    | **1 (cima)** | **1 (cima)** | **1 (cima)** | **1 (cima)** | **1 (cima)** |

##### Roteiro para testar na placa:

1. Suba `SW8`, `SW7`, `SW6` e `SW5` (demais para baixo) → monta 16384.
2. Aperte e solte `KEY0` para guardar como operando A.
3. Abaixe `SW8` a `SW5` e suba todas as chaves da fração (`SW4` a `SW0`) → representa 0,984375 como operando B.
4. O resultado aparece direto nos displays.

##### A conta passo a passo (os 4 estágios):

Com A = 16384 (exp 15, frac `10000000`) e B = 0,984375 (exp 0, frac `11111100`):

1. **Ordenação:** A tem expoente muitíssimo maior, então vira o "número grande"; B é o "pequeno".
2. **Alinhamento:** `exp_diff = 15 − 0 = 15`. O deslocamento necessário é maior do que os 8 bits da fração de B: ela desliza inteira para fora e `fraca` vira `00000000`. O operando pequeno é completamente descartado nesta etapa, antes mesmo da soma.
3. **Soma:** sinais iguais → soma: `10000000 + 00000000 = 10000000` (128), sem carry.
4. **Normalização:** a fração já começa com `1` → nenhum deslocamento. `expn = expb = 15`.

Resultado: sinal `+`, expoente **15**, fração **128** (`0x80`), exatamente igual ao valor original de A. Somar 0,984 a 16384 não mudou nada visível no resultado, porque a diferença de magnitude é grande demais para o formato de 13 bits capturar.

##### Lendo o resultado nos displays:

| Display | Mostra                   | Neste exemplo      |
| ------- | ------------------------ | ------------------ |
| `HEX3`  | sinal                    | apagado (positivo) |
| `HEX2`  | expoente, em hexadecimal | `F`                |
| `HEX1`  | 4 bits altos da fração   | `8`                |
| `HEX0`  | 4 bits baixos da fração  | `0`                |

![Caso 5: 16384+0,984, operando engolido, testado na placa](../doc/img//caso5_16384mais0p984_engolido.png)

---

#### Caso 6: 132 − 128 = 4 (deslocamento de 5 bits na normalização)

Este caso testa o segmento de contagem/deslocamento de zeros à esquerda no 4º estágio, usado quando a subtração produz um resultado com vários bits mais significativos zerados. Escolhemos um par de números com deslocamento maior (5 bits) para deixar bem evidente o "muito deslocamento" pedido pela professora nesta etapa.

##### Escrevendo os números no formato de 13 bits:

**Para o 132:**

- Primeira potência de 2 acima de 132 → `2^8 = 256`, então expoente = 8 (`1000`).
- `132 / 256 = 0,515625`; `× 256 = 132` → fração = 132 = `10000100`.

**Para o 128:**

- Primeira potência de 2 acima de 128 → `2^8 = 256`, então expoente = 8.
- `128 / 256 = 0,5`; `× 256 = 128` → fração = 128 = `10000000`.

Como é uma subtração (132 − 128), o 128 entra com o bit de sinal ligado (negativo): o somador soma quando os sinais são iguais e subtrai quando são diferentes, então isso equivale a calcular `132 + (−128)`.

##### Quais chaves `SW` acionar:

| Operando | Valor | `SW9` (sinal) | `SW8`        | `SW7`     | `SW6`     | `SW5`     | `SW4`     | `SW3`     | `SW2`     | `SW1`     | `SW0`        |
| -------- | ----- | ------------- | ------------ | --------- | --------- | --------- | --------- | --------- | --------- | --------- | ------------ |
| A        | +132  | 0 (baixo)     | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | **1 (cima)** |
| B        | −128  | **1 (cima)**  | **1 (cima)** | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo) | 0 (baixo)    |

##### Roteiro para testar na placa:

1. Suba `SW8` e `SW0` (demais para baixo) → monta 132.
2. Aperte e solte `KEY0` para guardar como operando A.
3. Suba também `SW9` (sinal negativo), mantenha `SW8` levantado e abaixe `SW0` → representa −128 como operando B.
4. O resultado (4) aparece direto nos displays.

##### A conta passo a passo (os 4 estágios):

Com A = 132 (exp 8, frac `10000100`, sinal +) e B = 128 (exp 8, frac `10000000`, sinal −):

1. **Ordenação:** mesmo expoente, então o circuito compara as frações: `132 > 128`, logo A (132) vira o "número grande" (`b`) e B (128) o "número pequeno" (`s`).
2. **Alinhamento:** `exp_diff = 8 − 8 = 0`, nenhuma fração desliza.
3. **Subtração:** sinais diferentes → subtrai: `10000100` (132) `− 10000000` (128) `= 00000100` (4), sem carry (`sum(8) = 0`).
4. **Normalização:** contando os zeros à esquerda em `00000100`, o primeiro `1` só aparece no bit 2 → **`leado = 5`**. Como `leado (5)` não é maior que `expb (8)`, é o caso normal: o circuito desloca a fração **5 bits** à esquerda e decrementa o expoente: `expn = 8 − 5 = 3`; `fracn = 10000000` (128).

Resultado: sinal `+` (herdado do operando de maior magnitude, 132), expoente **3**, fração **128** (`10000000` = `0x80`). Em decimal: `128 / 256 × 2^3 = 0,5 × 8 = 4`.

##### Lendo o resultado nos displays:

| Display | Mostra                   | Neste exemplo      |
| ------- | ------------------------ | ------------------ |
| `HEX3`  | sinal                    | apagado (positivo) |
| `HEX2`  | expoente, em hexadecimal | `3`                |
| `HEX1`  | 4 bits altos da fração   | `8`                |
| `HEX0`  | 4 bits baixos da fração  | `0`                |

![Caso 6: 132−128, deslocamento de 5 bits, testado na placa](../doc/img//caso6_132menos128_muitodeslocamento.png)

---
