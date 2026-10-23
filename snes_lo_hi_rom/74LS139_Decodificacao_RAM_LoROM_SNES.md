# 74LS139 no cartucho SNES LoROM — decodificação de RAM

## 1. Para que serve o 74LS139?

O **74LS139** é um circuito integrado com **dois decodificadores independentes de 2 entradas para 4 saídas** (dual 2-to-4 decoder). Cada metade possui:

- Uma entrada de habilitação ativa em nível baixo, **`/E`** (ou `/G`);
- Duas entradas de seleção (`A0` e `A1`);
- Quatro saídas ativas em nível baixo (`/O0` a `/O3`).

Ele não armazena dados nem converte por si só os endereços do SNES em offsets da ROM. Sua função é reconhecer combinações de sinais e ativar seletivamente uma saída. No esquema mostrado do Mouse Bite Labs, as duas metades **U5B e U5A são usadas em cascata para gerar a seleção da RAM**.

> **Convenção:** neste documento `/SINAL` significa *ativo em LOW* (nível lógico 0). Na imagem, os sinais aparecem nomeados `ROMSEL` e `RAM.CE`; a polaridade é determinada pelos pinos e pela lógica do circuito.

## 2. Tabela-verdade de uma metade do 74LS139

| `/E` | A1 | A0 | Saída em LOW | Outras saídas |
|---:|---:|---:|---|---|
| 1 | X | X | Nenhuma | Todas HIGH |
| 0 | 0 | 0 | `/O0` | HIGH |
| 0 | 0 | 1 | `/O1` | HIGH |
| 0 | 1 | 0 | `/O2` | HIGH |
| 0 | 1 | 1 | `/O3` | HIGH |

`X` = qualquer valor. As saídas não selecionadas permanecem HIGH.

## 3. Primeiro estágio: U5B (metade superior)

No recorte do esquemático:

| Pino físico | Sinal no símbolo | Conexão | Papel |
|---:|---|---|---|
| 15 | E (ativo LOW) | `/ROMSEL` | Habilita U5B |
| 14 | A0 | `A15` do SNES | Seleção |
| 13 | A1 | `A15` do SNES | Mesma seleção |
| 12 | O0 (ativo LOW) | E de U5A (pino 1) | Habilita segundo estágio |
| 11 | O1 | Não conectado | — |
| 10 | O2 | Não conectado | — |
| 9 | O3 | Não conectado | — |

Como **A0 e A1 recebem o mesmo sinal `A15`**, somente as combinações `00` e `11` são possíveis. A saída `/O0` fica LOW apenas quando `/ROMSEL = 0` **e** `A15 = 0`.

| `/ROMSEL` | A15 | Entrada A1:A0 | `/O0` de U5B |
|---:|---:|---|---:|
| 1 | 0 | 00 | 1 |
| 1 | 1 | 11 | 1 |
| 0 | 0 | 00 | **0** |
| 0 | 1 | 11 | 1 |

Isso serve para permitir o próximo estágio apenas na **metade inferior** de um banco (`$0000–$7FFF`, A15 = 0), quando `/ROMSEL` estiver ativo.

**Importante:** `/ROMSEL` é um sinal de seleção do barramento do cartucho do SNES; seu nome não significa que ele só possa participar da seleção da ROM. Sua interpretação exata depende da decodificação interna do console.

## 4. Segundo estágio: U5A (metade inferior)

| Pino físico | Sinal no símbolo | Conexão | Papel |
|---:|---|---|---|
| 1 | E (ativo LOW) | `/O0` de U5B | Habilita U5A |
| 2 | A0 | `BA6` | Bit 6 do banco SNES |
| 3 | A1 | `BA5` | Bit 5 do banco SNES |
| 4 | O0 | Não conectado | — |
| 5 | O1 | Não conectado | — |
| 6 | O2 | Não conectado | — |
| 7 | O3 | `RAM.CE` | Seleção ativa em LOW da RAM |

Quando U5A está habilitado (`E = 0`):

| BA6 (A0) | BA5 (A1) | Saída em LOW |
|---:|---:|---|
| 0 | 0 | O0 |
| 1 | 0 | O1 |
| 0 | 1 | O2 |
| 1 | 1 | **O3 = `RAM.CE`** |

Assim, `/RAM.CE` é acionado quando:

```text
/ROMSEL = 0
A15     = 0
BA6     = 1
BA5     = 1
```

Em termos de condição de ativação:

```text
RAM selecionada = (NOT /ROMSEL) AND (NOT A15) AND BA6 AND BA5
```

A expressão acima é a condição booleana de **seleção**, não a polaridade elétrica do fio `/RAM.CE`: quando a condição é verdadeira, `/RAM.CE` vai para **0**.

## 5. Exemplos no mapa de memória

Considerando os sinais de seleção do SNES em acessos típicos aos bancos correspondentes:

| Endereço da CPU | A15 | BA6 | BA5 | Resultado esperado |
|---|---:|---:|---:|---|
| `$00:8000` | 1 | 0 | 0 | RAM desabilitada; acesso usual à ROM |
| `$70:1234` | 0 | 1 | 1 | **RAM selecionada** |
| `$70:9000` | 1 | 1 | 1 | RAM desabilitada |
| `$60:1234` | 0 | 1 | 1 | **RAM selecionada** |
| `$7E:1234` | 0 | 1 | 1 | RAM não selecionada no uso normal: região de WRAM do console, não um acesso ao cartucho com `/ROMSEL` ativo |

**Por que `$60:1234` também pode selecionar RAM?** Porque BA5 e BA6 são iguais a 1 tanto nos bancos `$60–$7F` quanto em `$70–$7F`; BA4 **não entra** nessa etapa de decodificação. O acesso efetivo também depende do sinal `/ROMSEL` gerado pelo console.

A tabela descreve a lógica mostrada, não garante que qualquer jogo enxergará RAM em toda essa faixa: isso também depende do mapeamento implementado no console e das demais ligações da placa.

## 6. Isso também controla a ROM?

**Não diretamente nesse trecho.** No recorte do esquemático:

- **U5B** reconhece a condição `/ROMSEL = 0` e `A15 = 0`;
- **U5A** verifica `BA6` e `BA5` e produz o sinal **`RAM.CE`**;
- A linha **`ROM.OE`** passa pelo circuito, mas **não está conectada às saídas desses dois decodificadores**.

Não se deve concluir, apenas por esse recorte, de onde vem `/CE` ou `/OE` da ROM: é necessário seguir essas redes nas outras páginas do esquema.

## 7. Aplicação ao projeto MX29F1610A + FM18W08

No cartucho planejado:

- A **MX29F1610A** fornece os dados do jogo em modo de 8 bits;
- A **FM18W08** armazena os saves sem bateria;
- Um decodificador como o **74HCT139** pode ajudar a reconhecer o espaço de endereços reservado ao save;
- A lógica precisa impedir que Flash e FeRAM **dirijam D0–D7 ao mesmo tempo**.

**Atenção especial à FeRAM:** a FM18W08 não deve necessariamente receber, sem adaptação, o mesmo `/CE`, `/OE` e `/WE` usado por uma SRAM convencional. É preciso conferir a temporização de captura de endereço, leitura e escrita no datasheet e no circuito de referência SNES-FeRAM-Cart.

### Por que considerar 74HCT139 no lugar de 74LS139?

Ambos executam a mesma função lógica, mas usam famílias elétricas diferentes. O **74HCT139**, com alimentação de 5 V, apresenta entradas compatíveis com níveis TTL e normalmente é uma escolha interessante em um projeto novo. A escolha final requer verificar corrente de saída, fan-out, atrasos e disponibilidade.

## 8. Resumo

> No esquema analisado, o 74LS139 funciona como um **detector de região de save**. U5B verifica `/ROMSEL` e A15; sua saída habilita U5A, que verifica BA6 e BA5. Quando tudo coincide, a saída `RAM.CE` vai a LOW e seleciona a RAM. A habilitação da ROM não é gerada diretamente por esse trecho.

## Referências

- [Mouse Bite Labs — The SNES Cartridge, Briefly Explained](https://mousebitelabs.com/2019/05/18/custom-pcb-explanation/)
- [MouseBiteLabs — Super Nintendo Cartridges](https://github.com/MouseBiteLabs/Super-Nintendo-Cartridges)
- [SNES-FeRAM-Cart — soniccd123](https://github.com/soniccd123/SNES-FeRAM-Cart)

*Material de estudo baseado no trecho de esquemático fornecido. As conexões completas devem ser verificadas no projeto KiCad antes de fabricar a placa.*
