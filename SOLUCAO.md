# Solução — Assistente de Investimentos com RPA e n8n

Entrega do desafio de projeto do bootcamp **Santander 2026 — Automação com N8N** (DIO).

Este documento descreve **o que foi implementado e por que cada decisão foi tomada**. O código
está em [`n8n/workflow.json`](n8n/workflow.json) e em
[`rpa/extrair_clientes.ipynb`](rpa/extrair_clientes.ipynb).

---

## O que o fluxo faz

```
GitHub Pages ──HTTP──> RPA (Python + BeautifulSoup)
                              │
                              │ POST JSON
                              ▼
                        Webhook (n8n)
                              │
          ┌───────────────────┴───────────────────┐
          │ Input 1: clientes                      │
          │                                        │
   HTTP Request (data.csv) ──> Parse do CSV ──> Merge ──> Cruzar e Gerar
                                   Input 2                 Mensagens
                                                              │
                                                     Respond to Webhook
```

O RPA lê a tabela de clientes de uma página HTML, envia a lista para o n8n, que busca a tabela de
produtos, cruza cada cliente com as opções do seu perfil e devolve uma mensagem pronta por cliente.

**Saída medida:** 10 clientes, 9 produtos, 10 recomendações geradas.

---

## Decisões técnicas

### 1. A comparação saldo × mínimo é numérica, não textual

O saldo vem da página como `"R$ 12.500,00"` e o mínimo vem do CSV como `"1000"`. Comparar os dois
como texto parece funcionar e erra em silêncio.

A conversão trata a ordem dos separadores, que é onde a maioria das implementações quebra:

```js
.replace(/[^\d.,-]/g, "")  // remove R$, espaços
.replace(/\./g, "")        // ponto é separador de MILHAR
.replace(",", ".")         // vírgula é decimal
```

Inverter essas duas linhas transforma `12.500,00` em `12,5` — o cliente passa a não ter saldo para
nada, e o fluxo continua rodando sem erro nenhum.

### 2. Três situações, não duas

A implementação ingênua é "tem saldo / não tem saldo". O fluxo distingue **três**, e cada uma gera
mensagem diferente:

| `motivo` | Quando | O que a mensagem faz |
| --- | --- | --- |
| `saldo_ok` | existe produto do perfil que o saldo alcança | recomenda o produto |
| `saldo_insuficiente` | existe produto do perfil, mas nenhum alcançado | aponta o de menor entrada **como meta** |
| `perfil_sem_produto` | não há produto cadastrado para o perfil | encaminha para atendimento humano |

⚠️ **Nenhum cliente é descartado em silêncio.** Descartar sem registrar é o jeito mais comum de um
fluxo automatizado perder trabalho sem ninguém perceber.

### 3. Entre os produtos acessíveis, escolhe o de maior mínimo

Dentro do perfil, quanto maior a aplicação mínima, mais completo tende a ser o produto. Como o
cliente já tem saldo para ele, recomendar o mais barato subaproveitaria a carteira.

### 4. A escolha da mensagem é determinística

São geradas três variações por cliente, mas a escolhida é sempre a primeira. **Rodar duas vezes com
a mesma entrada produz a mesma saída** — sem isso não há como testar o fluxo nem comparar execuções.

Sorteio aleatório daria variedade e tiraria a reprodutibilidade. Para um MVP, reprodutibilidade vale
mais.

### 5. O resultado traz um agregado, não só a lista

Além de `resultados`, a resposta devolve:

```json
"resumo_por_motivo": { "saldo_ok": 10 }
```

Uma lista de 10 itens se lê; uma de 10.000 não. O agregado é o que permite perceber, de relance, que
o fluxo passou a classificar metade dos clientes como `perfil_sem_produto` — sintoma de que o CSV
mudou ou de que um perfil novo apareceu na origem.

### 6. Os nós de código falham alto

Tanto o parse do CSV quanto o cruzamento lançam erro explícito quando recebem entrada vazia:

```js
if (clientes.length === 0) {
  throw new Error("Nenhum cliente recebido. Confira se o POST do RPA chegou no Webhook.");
}
```

Sem isso, um webhook mal configurado produz uma execução verde com zero resultados — que é pior do
que falhar, porque parece sucesso.

---

## Como executar

1. **n8n** — importar `n8n/workflow.json`
   O webhook responde em `POST /webhook/clientes`.
   Rodando localmente, subir com `n8n start --tunnel` para obter uma URL pública, já que o Colab
   roda na nuvem e não alcança `localhost`.

2. **Copiar a Production URL** do nó Webhook
   ⚠️ A *Test URL* só responde enquanto o botão "Listen for test event" está ativo.

3. **Colab** — abrir `rpa/extrair_clientes.ipynb`, colar a URL em `N8N_WEBHOOK` e executar.

O notebook valida a URL antes de rodar e interrompe com mensagem clara se ela não foi preenchida.

---

## Escopo

Implementado o **MVP** do desafio: fluxo completo com **mensagens estáticas por template**.

A geração via LLM e o agente de IA constam como **desafio extra** e não foram incluídos — o fluxo
foi desenhado para recebê-los sem reescrita: basta substituir a função `montarMensagens` por uma
chamada ao modelo, mantendo o mesmo contrato de entrada e saída.
