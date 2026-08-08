# Tutorial: Implementação de Somador Ponto Flutuante na DE10 Lite

**Autores:** Caique Castro Rodrigues, Henrique, Gustavo

**Disciplina:** Sistemas Digitais Q2.2026

**Data:** 09/08/2026

---

## 1. Objetivo do Projeto

Este projeto adapta o **somador de ponto flutuante simplificado de 13 bits** do livro didático (Pong P. Chu, *FPGA Prototyping by VHDL Examples*, Listings 3.19 e 3.20) para a placa **Terasic DE10 Lite**, que usa a FPGA **Intel MAX 10 (modelo `10M50DAF484C7G`)**. A ideia é passar por todo o caminho: **sintetizar** e **simular** o circuito em VHDL e, no fim, rodar ele na placa de verdade.

### Formato numérico de 13 bits

O número é representado em três campos:

| Campo | Bits | Significado |
|---|---|---|
| `s` | 1 | sinal (0 = positivo, 1 = negativo) |
| `e` | 4 | expoente, sem sinal (*unsigned*) |
| `f` | 8 | significando (fração pura `0.f`) |

O valor representado é:

$$\text{valor} = (-1)^{s} \times 0.f \times 2^{e}$$
