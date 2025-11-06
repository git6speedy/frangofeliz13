# Correção de Erros - Módulo Financeiro

## Data: 06/11/2025

### Problemas Encontrados

#### 1. Erro na Migração - Índice Duplicado
**Erro:**
```
ERROR: 42P07: relation "idx_financial_categories_store" already exists
```

**Causa:** 
A migração `20250106_create_financial_module.sql` estava tentando criar índices sem verificar se já existiam, causando erro ao rodar a migração múltiplas vezes.

**Solução:**
Alterados todos os comandos `CREATE INDEX` para `CREATE INDEX IF NOT EXISTS` em todas as tabelas do módulo financeiro:
- financial_categories
- bank_accounts
- credit_cards
- financial_transactions
- accounts_receivable
- dream_board
- financial_goals
- financial_notifications

#### 2. Erro na Query PostgREST - Relacionamento Ambíguo
**Erro:**
```
PGRST201: Could not embed because more than one relationship... for 'financial_transactions' and 'bank_accounts'
```

**Causa:**
A tabela `financial_transactions` possui dois foreign keys para `bank_accounts`:
- `bank_account_id` - Conta de origem/destino principal
- `transfer_to_account_id` - Conta de destino em transferências

Ao fazer a query com `.select('bank_account:bank_accounts(*), transfer_to_account:bank_accounts(*)')`, o PostgREST não sabia qual foreign key usar.

**Solução:**
Especificado explicitamente o nome da coluna foreign key na query usando a sintaxe `!column_name`:

```typescript
// ANTES:
.select(`
  *,
  category:financial_categories(*),
  bank_account:bank_accounts(*),
  credit_card:credit_cards(*),
  transfer_to_account:bank_accounts(*)
`)

// DEPOIS:
.select(`
  *,
  category:financial_categories(*),
  bank_account:bank_accounts!bank_account_id(*),
  credit_card:credit_cards(*),
  transfer_to_account:bank_accounts!transfer_to_account_id(*)
`)
```

### Arquivos Modificados

1. **supabase/migrations/20250106_create_financial_module.sql**
   - Adicionado `IF NOT EXISTS` em todos os `CREATE INDEX`

2. **src/hooks/useFinancialTransactions.ts**
   - Especificado foreign keys explícitas na query PostgREST

### Como Testar

1. **Migração:**
   ```sql
   -- Rode a migração novamente - agora não deve dar erro
   -- Execute no SQL Editor do Supabase
   ```

2. **Query de Transações:**
   - Acesse a página `/financas` (após login)
   - Navegue até a aba "Lançamentos"
   - Verifique que as transações carregam corretamente
   - No console do navegador, não deve aparecer mais o erro PGRST201

### Observações

- A migração agora é **idempotente** - pode ser executada múltiplas vezes sem erros
- A query PostgREST agora funciona corretamente com múltiplos relacionamentos para a mesma tabela
- O padrão `table:foreign_table!foreign_key_column(*)` deve ser usado sempre que houver múltiplas FKs para a mesma tabela
