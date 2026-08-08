## 5. Diário de Bordo de IA

Utilizamos o **Claude** como ferramenta de apoio ao longo do
projeto, principalmente para acelerar a escrita do testbench, apoiar a
refatoração do circuito de teste e tirar dúvidas sobre o algoritmo do livro.

---

### 5.1 Como conduzimos o uso da ferramenta

Não pedimos "faça o projeto", primeiro entendemos o
artigo (Pong P. Chu, seção 3.7.4), depois usamos a IA para checar nosso
entendimento, gerar rascunhos e, principalmente, para nos forçar a justificar
cada decisão. além de termos utilizado para nos auxiliar no uso do questa, uma vez que nenhum integrante do grupo tinha muita familiaridade com o sistema. 

#### Prompts utilizados (seleção)

Registramos abaixo os prompts mais representativos, na ordem em que a
investigação evoluiu. A conversa completa está anexada em PDF.

> "Explique o 4º estágio (normalização) do somador de ponto flutuante do
> Listing 3.19 do Pong P. Chu. Por que existem três condições diferentes
> (`sum(8)='1'`, `leado > expb`, e o caso normal)? Quero entender o que cada
> uma representa fisicamente antes de mexer no hardware."

> "Compilamos o VHDL do PDF e ele não passou no GHDL. Os erros são em `'0 '` e
> `= >`. Isso é erro de lógica ou de extração do PDF? Como confirmar que a
> lógica está intacta?"

> "A placa do livro tinha 4 displays multiplexados e a DE10-Lite tem 6
> independentes. Faz sentido remover o módulo `disp_mux`? Quais consequências
> isso tem para o resto do circuito de teste?"

> "Queremos um testbench no mesmo formato do Lab 2 (gerado pelo Test Bench
> Template Writer do Quartus, com processos `init` e `always`), mas
> auto-verificável, que compare cada saída com o valor esperado e conte os
> erros. Como estruturar? Quais os passos podemos seguir no Questa desde o teste até conectar na placa?"

> "Quais casos de teste cobrem as três condições de normalização descritas na
> seção 3.7.4 do texto? Queremos que cada caso rastreie para um exemplo ou
> para uma decisão nossa, não escolher números no olho."

> "Com todas as chaves para baixo, o que os LEDs deveriam mostrar? Queremos
> um teste para saber se a gravação funcionou."

> "Precisamos remover a memória do operando A dos displays
> (HEX1/HEX0) e usar só 4 displays. O que precisa mudar no `.vhd` e no
> testbench para não quebrar a validação?"

>"Como deixar nosso relatório em Markdown mais bonito e legível para o GitHub? queremos só a apresentação mais limpa."

>"Queremos representar os fluxos do projeto com diagramas Mermaid embutidos no Markdown (que o GitHub renderiza sozinho), em vez de imagens soltas. Precisamos de: (1) um diagrama de blocos do fp_adder mostrando os dois operandos entrando e o resultado saindo; (2) um fluxograma dos 4 estágios internos (ordenação → alinhamento → soma/subtração → normalização), com a decisão final da normalização ramificando em carry-out, underflow e caso normal; (3) um diagrama do top-level da placa ligando SW/KEY/clock aos displays e LEDs. Gere o código Mermaid e explique para conseguirmos ajustar depois."

---

### 5.2 Os erros da IA (alucinações) e como corrigimos

#### Erro 1 — Pinos e padrão elétrico assumidos de memória

**O que a IA fez:** ao gerar o primeiro rascunho do arquivo de pinos
(`.tcl`), a IA preencheu as localizações (`PIN_C14`, `PIN_C10`, etc.) a partir
do que ela tinha de conhecimento, sem citar a página do manual. Em uma das versões, ela
sugeriu o I/O Standard como `"3.3-V LVTTL"` para todos os pinos, incluindo
os botões `KEY`.

**A auditoria que fizemos:** conferimos pino a pino contra o *DE10-Lite
User Manual* oficial da Terasic (Tabela 3-17 e tabelas de pinagem dos
displays, chaves, botões e LEDs, deixamos o manual no repositório dentro da pasta "/doc/files"). Resultado da auditoria:

| Item auditado | O que o manual diz | O que estava no `.tcl` | Situação |
|---|---|---|---|
| `HEX0[0]` | PIN_C14, "Seven Segment Digit 0[0]", 3.3-V LVTTL | PIN_C14 | ✅ confere |
| `HEX0[7]` | PIN_D15, "Digit 0[7], **DP**" (ponto decimal) | PIN_D15 | ✅ confere |
| `SW[0]` | PIN_C10 | PIN_C10 | ✅ confere |
| `KEY[0]` | PIN_B8 | PIN_B8 | ✅ confere |
| I/O Standard geral | **3.3-V LVTTL** | 3.3-V LVTTL | ✅ confere |
| Polaridade dos displays | **anodo comum: segmento acende com nível BAIXO** | tratado no decodificador | ✅ confere |

A pinagem em si estava correta, mas só soubemos disso porque conferimos,
não porque a IA garantiu.

#### Erro 2 — *Default binding* que quebrava só no GHDL

**O que aconteceu:** o primeiro testbench compilava e rodava no Questa, mas no
GHDL parava com um erro de *binding* (a instância do componente não achava a
entidade real). A IA havia omitido a especificação de configuração, porque o
Questa faz esse *bind* automaticamente e ela "assumiu" que bastava.

**A correção humana:** identificamos que faltava ligar explicitamente o
`COMPONENT` à `ENTITY`. Acrescentamos, logo após o `END COMPONENT;`, a linha:

```vhdl
FOR ALL : fp_adder_de10lite USE ENTITY work.fp_adder_de10lite(arch);
```

#### Erro 3 — Confusão entre "Analysis & Elaboration" e "Analysis & Synthesis"

**O que aconteceu:** seguindo o roteiro, rodamos *Start Analysis & Elaboration*
(como no Lab 2) e, ao tentar o *Partition Merge* para gerar o template do
testbench, o Quartus abortou com:

```
Error (39003): Run Analysis and Synthesis (quartus_map) ... before running
Compiler Database Interface (quartus_cdb)
```

**A correção humana:** entendemos a diferença entre os dois comandos. A
*Elaboration* só monta a hierarquia; ela **não gera a netlist sintetizada**, e
o *Partition Merge* (`quartus_cdb`) precisa dessa netlist. A solução foi rodar
**Start Analysis & Synthesis** (`Ctrl+K`) antes do Partition Merge. Documentamos
o erro e a causa no nosso passo a passo, porque é um tropeço fácil para quem
vem do Lab 2 (onde a Elaboration bastava).

---

### 5.4 Avaliação crítica da ferramenta

**Onde ajudou de verdade:** acelerou a escrita do testbench e na retirada de dúvidas sobre o sistema. Também ajudou na apresentação da documentação: a formatação
do Markdown e os diagramas em Mermaid.

**Onde não se pode confiar:** em qualquer fato específico da placa (pinos,
polaridade, padrão elétrico) e em qualquer coisa que dependa do ambiente
(diferença entre comandos do Quartus, *binding* entre simuladores). Nesses
pontos, a IA erra com confiança, e **só a documentação oficial e o teste real
resolvem**. Foi exatamente onde ela errou (Erros 1 a 3) que tivemos que
estudar mais a fundo.

**Conclusão do grupo:** a ferramenta foi útil como acelerador e como
interlocutor para checar entendimento, mas **não substitui a leitura do artigo,
a auditoria contra o manual e o teste na bancada**.

---