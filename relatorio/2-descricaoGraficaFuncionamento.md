## 2. Descrição gráfica do funcionamento do sistema

Nesta etapa, a descrição gráfica se refere ao **código VHDL do somador apresentado no artigo** (Pong P. Chu, Listing 3.19, a entidade `fp_adder`). Aqui usamos exatamente as **portas do artigo**, ainda sem nada específico da placa; a versão com os pinos da DE10 Lite aparece na Parte 3.

O `fp_adder` recebe **dois números em ponto flutuante** (cada um com sinal, expoente e fração) e devolve **um único resultado**, também com sinal, expoente e fração.

### 2.1 Diagrama de blocos do somador `fp_adder`

```mermaid
flowchart LR
    subgraph OP1["Operando 1"]
        A1["sign1 (1 bit)<br/>exp1 (4 bits)<br/>frac1 (8 bits)"]
    end

    subgraph OP2["Operando 2"]
        A2["sign2 (1 bit)<br/>exp2 (4 bits)<br/>frac2 (8 bits)"]
    end

    subgraph CORE["fp_adder"]
        ADD["Somador de ponto flutuante<br/>4 estágios"]
    end

    subgraph RES["Resultado"]
        R["sign_out (1 bit)<br/>exp_out (4 bits)<br/>frac_out (8 bits)"]
    end

    A1 --> ADD
    A2 --> ADD
    ADD --> R
```

### 2.2 Fluxo interno do núcleo `fp_adder` (4 estágios)

O núcleo é **combinacional** e reproduz, em hardware, o que se faz no papel ao somar em notação científica.

```mermaid
flowchart TD
    IN["Entradas:<br/>sign1/exp1/frac1<br/>sign2/exp2/frac2"] --> E1

    E1["1º estágio: Ordenação (sort)<br/>compara exp &amp; frac e separa<br/>maior (b) e menor (s)"] --> E2

    E2["2º estágio: Alinhamento (align)<br/>exp_diff = expb − exps<br/>desloca fracs à direita → fraca"] --> E3

    E3["3º estágio: Soma/Subtração<br/>sinais iguais → soma<br/>sinais diferentes → subtração<br/>resultado em sum(8..0)"] --> E4

    E4["4º estágio: Normalização<br/>leado = conta zeros à esquerda<br/>sum_norm = desloca à esquerda"] --> DEC

    DEC{"Decisão final"} -->|"sum(8) = 1 (carry out)"| C1["expn = expb + 1<br/>fracn = sum(8..1)"]
    DEC -->|"leado > expb (underflow)"| C2["expn = 0<br/>fracn = 0 (força zero)"]
    DEC -->|"caso normal"| C3["expn = expb − leado<br/>fracn = sum_norm"]

    C1 --> OUT["Saídas:<br/>sign_out / exp_out / frac_out"]
    C2 --> OUT
    C3 --> OUT
```

### 2.3 Sinais de entrada

Cada operando entra no somador com três campos, no formato de 13 bits do artigo:

| Sinal | Largura | Função |
|---|---|---|
| `sign1` | 1 bit | sinal do operando 1 (`0` = positivo, `1` = negativo) |
| `exp1` | 4 bits | expoente do operando 1 |
| `frac1` | 8 bits | significando (fração `0.f`) do operando 1 |
| `sign2` | 1 bit | sinal do operando 2 |
| `exp2` | 4 bits | expoente do operando 2 |
| `frac2` | 8 bits | significando (fração `0.f`) do operando 2 |

### 2.4 Sinais de saída

O resultado sai no mesmo formato de 13 bits:

| Sinal | Largura | Função |
|---|---|---|
| `sign_out` | 1 bit | sinal do resultado |
| `exp_out` | 4 bits | expoente do resultado |
| `frac_out` | 8 bits | significando (fração `0.f`) do resultado |

---