# 📦 Funcionalidade: Itens Compostos

## 🎯 Objetivo

Permitir que variações de produtos sejam derivadas de outros produtos (matéria-prima), com controle automático de estoque. Exemplo: um Frango Assado Inteiro pode gerar 2 Meios Frangos.

---

## 🚀 Como Usar

### 1️⃣ Cadastrar um Produto com Variações

1. Acesse **Produtos**
2. Adicione um produto marcando **"Possui Variações?"** = SIM
3. Exemplo: Produto = "Frango Assado Recheado"

### 2️⃣ Criar uma Variação como Item Composto

1. Clique em **Gerenciar Variações** no produto criado
2. Preencha o nome da variação: "Meio Frango Assado Recheado"
3. Marque **"Este é um Item Composto?"** = SIM
4. Configure:
   - **Matéria-Prima**: Selecione um produto OU uma variação de outro produto
     - **Produtos** aparecem com ícone 📦
     - **Variações** aparecem com ícone 🔸 e mostram: Nome do Produto - Nome da Variação
   - **Rendimento**: Digite 2 (quantas unidades gera)
5. Salve a variação

### Exemplo com Variações:

**Produto: Farofa**
- Variação 1: Farofa de Bacon
  - É composto: SIM
  - Matéria-prima: 🔸 Bacon - Bacon Picado (variação)
  - Rendimento: 3
  
- Variação 2: Farofa de Pão
  - É composto: SIM
  - Matéria-prima: 📦 Pão Amanhecido (produto)
  - Rendimento: 5

---

## 📋 Como Funciona

### Ao Vender um Item Composto:

**Regra Principal:** O sistema SEMPRE consome primeiro o estoque do produto composto. Só consome matéria-prima se o estoque estiver insuficiente.

**Exemplo 1:** Cliente compra 1 "Meio Frango Assado Recheado" (COM estoque = 5)

1. ✅ Sistema verifica que é um item composto
2. ✅ Verifica que há estoque suficiente (5 unidades)
3. ✅ Consome 1 unidade do estoque do produto composto
4. ✅ Estoque final da variação = 4 unidades
5. ✅ Matéria-prima NÃO é consumida

**Exemplo 2:** Cliente compra 3 "Meio Frango Assado Recheado" (COM estoque = 1)

1. ✅ Sistema verifica que é um item composto
2. ✅ Verifica que NÃO há estoque suficiente (1 < 3)
3. ✅ Consome 1 unidade do estoque do produto composto
4. ✅ Faltam 2 unidades → consome matéria-prima
5. ✅ Consome 1 unidade da matéria-prima (2 vendas / 2 rendimento = 1)
6. ✅ Registra a transação para possível reversão

**Exemplo 3:** Cliente compra 1 "Meio Frango Assado Recheado" (SEM estoque = 0)

1. ✅ Sistema verifica que é um item composto
2. ✅ Verifica que NÃO há estoque (0)
3. ✅ Consome 1 unidade da matéria-prima (1 venda / 2 rendimento = arredonda para 1)
4. ✅ Registra a transação para possível reversão

### Cálculo de Consumo de Matéria-Prima:

**IMPORTANTE:** A matéria-prima só é consumida se não houver estoque suficiente do produto composto!

```
Se (Estoque do produto composto >= Quantidade vendida):
  → Consome APENAS do estoque do produto composto
  → Matéria-prima NÃO é consumida
Senão:
  → Quantidade que falta = Quantidade vendida - Estoque atual
  → Matéria-prima consumida = ARREDONDAR_PARA_CIMA(Quantidade que falta / Rendimento)
```

**Exemplos:**
- Estoque: 5 | Venda: 1 | Rendimento: 2 → Consome 0 matéria-prima (tem estoque)
- Estoque: 0 | Venda: 1 | Rendimento: 2 → Consome 1 matéria-prima (sem estoque)
- Estoque: 1 | Venda: 3 | Rendimento: 2 → Consome 1 matéria-prima (faltam 2 unidades)

---

## 🔄 Reversão em Cancelamentos (Futuro)

Quando um pedido com item composto for cancelado, o sistema:

1. Busca a transação registrada em `composite_item_transactions`
2. Restaura o estoque da matéria-prima
3. Ajusta o estoque da variação
4. Marca a transação como revertida

---

## 🎨 Interface

### Card de Item Composto

- **Switch** para ativar/desativar
- **Dropdown** para selecionar matéria-prima (mostra produtos sem variação)
- **Campo numérico** para definir o rendimento
- **Exemplo visual** explicando o funcionamento
- **Badge "Item Composto"** na listagem de variações
- **Informações** sobre matéria-prima e rendimento

### Validações

- ✅ Matéria-prima é obrigatória quando marcado como composto
- ✅ Rendimento mínimo = 1
- ✅ Não permite selecionar o próprio produto como matéria-prima
- ✅ Estoque inicial desabilitado (gerado automaticamente)

---

## 💾 Estrutura do Banco de Dados

### Tabela: `product_variations`

Novos campos:
- `is_composite` (BOOLEAN) - Se é um item composto
- `raw_material_product_id` (UUID) - ID da matéria-prima
- `yield_quantity` (INTEGER) - Quantidade gerada por unidade

### Tabela: `composite_item_transactions`

Registra cada venda de item composto:
- `order_id` - ID do pedido
- `order_item_id` - ID do item do pedido
- `variation_id` - ID da variação vendida
- `raw_material_product_id` - ID da matéria-prima consumida
- `raw_material_consumed` - Quantidade consumida
- `variations_generated` - Quantidade gerada
- `reversed_at` - Data da reversão (NULL se não revertido)

---

## 📊 Exemplo Completo

### Cadastro:

```
Produto: Frango Assado Recheado
├─ Estoque: 10 unidades
├─ Preço: R$ 35,00
└─ Variação: Meio Frango Assado Recheado
   ├─ Item Composto: SIM
   ├─ Matéria-prima: Frango Assado Recheado
   ├─ Rendimento: 2
   ├─ Ajuste de preço: R$ -15,00
   └─ Estoque: 0 (será gerado na venda)
```

### Venda no PDV (Cenário 1 - COM estoque):

```
Estoque inicial:
- Frango Assado Recheado: 10 unidades
- Meio Frango Assado Recheado: 3 unidades

Cliente compra: 1x Meio Frango Assado Recheado (R$ 20,00)

Processamento automático:
1. Verifica que é item composto
2. Verifica que há estoque (3 >= 1)
3. Consome 1 unidade do estoque do Meio Frango

Resultado:
✅ Frango Assado Recheado: 10 unidades (NÃO CONSUMIU)
✅ Meio Frango Assado Recheado: 2 unidades
✅ Cliente recebeu 1 Meio Frango
✅ Transação NÃO registrada (usou estoque próprio)
```

### Venda no PDV (Cenário 2 - SEM estoque):

```
Estoque inicial:
- Frango Assado Recheado: 10 unidades
- Meio Frango Assado Recheado: 0 unidades

Cliente compra: 1x Meio Frango Assado Recheado (R$ 20,00)

Processamento automático:
1. Verifica que é item composto
2. Verifica que NÃO há estoque (0 < 1)
3. Consome 1 Frango Assado Recheado (1 venda / 2 rendimento = arredonda para 1)

Resultado:
✅ Frango Assado Recheado: 9 unidades
✅ Meio Frango Assado Recheado: 0 unidades (continua em 0, pois vendeu sem estoque)
✅ Cliente recebeu 1 Meio Frango
✅ Transação registrada para possível reversão
```

---

## ⚠️ Observações Importantes

1. **Prioridade de estoque:** O sistema SEMPRE consome primeiro o estoque do produto composto. A matéria-prima só é consumida quando não há estoque suficiente.
2. **PDV e Totem:** Podem vender produtos compostos mesmo sem estoque (nesse caso, consome da matéria-prima).
3. **CustomStore:** Só pode vender produtos que tenham estoque disponível (não afetado por esta funcionalidade).
4. **Planeje o rendimento cuidadosamente** - uma vez vendido, a transação é calculada com base nele.
5. **Cancelamentos** ainda precisam ser implementados manualmente por enquanto.

---

## 🔧 Ativação no Banco

Execute o SQL em `EXECUTAR_NO_SUPABASE.sql` no seu painel do Supabase para ativar esta funcionalidade.

---

## ✅ Critérios Atendidos

- ✅ Produto composto pode ser vendido mesmo com estoque = 0
- ✅ Matéria-prima é reduzida ao final da venda confirmada
- ✅ Estrutura preparada para reversão em cancelamentos
- ✅ UI amigável com explicações e exemplos
- ✅ Interface e lógica integradas com Supabase
- ✅ Comportamento de estoque consistente
- ✅ Fluxo de venda robusto

---

## 📝 Próximos Passos

1. Implementar reversão automática em cancelamentos de pedidos
2. Adicionar relatório de itens compostos vendidos
3. Dashboard com alertas de matéria-prima baixa
4. Histórico de transações de itens compostos
