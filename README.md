### DICIONÁRIO DE DADOS - CUPCAKES GOURMET

#### 1. Tabela: `cliente`
Armazena as informações dos usuários do sistema, englobando tanto os clientes que realizam compras na vitrine virtual quanto os administradores que gerenciam o negócio.

| Nome da Coluna | Tipo de Dado Físico | Restrições / Chaves | Descrição e Regra de Negócio |
| :--- | :--- | :--- | :--- |
| `id_cliente` | SERIAL | PK | Identificador único autoincremental do cliente no sistema. |
| `nome` | VARCHAR(120) | NOT NULL | Nome completo do usuário. Utilizado para personalização da interface e identificação na entrega. |
| `telefone` | VARCHAR(30) | NOT NULL | Telefone de contato do cliente. Fundamental para dupla validação de segurança no rastreamento de pedidos e comunicação na entrega. |
| `usuario` | VARCHAR(60) | NOT NULL, UNIQUE | Nome de usuário (login) escolhido para acesso ao sistema. Deve ser único para evitar conflitos de autenticação. |
| `senha` | VARCHAR(255) | NOT NULL | Hash criptografado da senha do usuário (ex: gerado via scrypt). Garante a segurança do acesso à conta. |
| `endereco` | VARCHAR(255) | NOT NULL | Endereço completo para o delivery dos cupcakes. Necessário para o cálculo de rotas do entregador. |
| `is_admin` | BOOLEAN | NOT NULL, DEFAULT FALSE | Flag de privilégio. Se `true`, concede acesso ao Painel Admin (gestão de cardápio e status de pedidos). Por padrão, todo novo cadastro é de cliente comum. |

**Especificações Técnicas da Tabela `cliente`:**
* **Chave Primária (PK):** `id_cliente`
* **Chaves Estrangeiras (FKs):** Nenhuma.
* **Restrições de Validação de Domínio:** A restrição `UNIQUE` em `usuario` impede a duplicidade de contas com o mesmo login.

#### 2. Tabela: `produto`
Gerencia o cardápio (catálogo) de cupcakes da loja, controlando informações de exibição na vitrine e níveis de disponibilidade.

| Nome da Coluna | Tipo de Dado Físico | Restrições / Chaves | Descrição e Regra de Negócio |
| :--- | :--- | :--- | :--- |
| `id_produto` | SERIAL | PK | Identificador único autoincremental do cupcake no catálogo. |
| `nome` | VARCHAR(120) | NOT NULL | Nome comercial do cupcake (ex: "Cupcake de Morango Supremo"). Exibido na vitrine virtual e no carrinho. |
| `descricao` | TEXT | NOT NULL, DEFAULT '' | Detalhamento dos ingredientes e características do sabor para atrair o cliente na vitrine. |
| `preco` | NUMERIC(10,2) | NOT NULL, CHECK | Valor unitário de venda do cupcake. Utilizado no cálculo em tempo real do carrinho. |
| `quantidade_estoque` | INTEGER | NOT NULL, DEFAULT 0 | Quantidade física disponível do sabor. Controla o bloqueio de vendas (itens esgotados) e usa `SELECT ... FOR UPDATE` para evitar concorrência. |
| `ativo` | BOOLEAN | NOT NULL, DEFAULT TRUE | Flag de *soft delete*. Permite que o admin desative um sabor que saiu de linha sem excluí-lo do banco de dados, mantendo a consistência dos pedidos antigos. |

**Especificações Técnicas da Tabela `produto`:**
* **Chave Primária (PK):** `id_produto`
* **Chaves Estrangeiras (FKs):** Nenhuma.
* **Restrições de Validação de Domínio:** `CHECK (preco >= 0)` - Garante que o sistema jamais aceite o cadastro de um cupcake com valor financeiro negativo, prevenindo quebras no cálculo do checkout.

#### 3. Tabela: `pedido`
Registra a consolidação de uma compra (checkout), armazenando o status de acompanhamento, os dados financeiros totais e o método de pagamento escolhido.

| Nome da Coluna | Tipo de Dado Físico | Restrições / Chaves | Descrição e Regra de Negócio |
| :--- | :--- | :--- | :--- |
| `id_pedido` | SERIAL | PK | Identificador único autoincremental do pedido. Utilizado pelo cliente na tela de rastreamento seguro. |
| `id_cliente` | INTEGER | FK, NOT NULL | Vincula o pedido ao cliente que o realizou. Permite a listagem no Histórico de Compras da conta do usuário. |
| `data_hora` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Carimbo de tempo do momento exato do fechamento do pedido. Utilizado para ordenação prioritária na fila de produção do Painel Admin. |
| `status` | VARCHAR(30) | NOT NULL, CHECK, DEFAULT 'Recebido' | Estado atual do pedido na operação. Atualizado pelo administrador para que o cliente acompanhe no rastreamento. |
| `forma_pagamento` | VARCHAR(30) | NOT NULL, CHECK | Opção de pagamento selecionada no checkout. Orienta se o motoboy deve levar a maquininha ou apenas validar a transferência. |
| `valor_total` | NUMERIC(10,2) | NOT NULL | Somatória final financeira da transação (Subtotal dos itens + Taxa de entrega). |
| `taxa_entrega` | NUMERIC(10,2) | NOT NULL | Custo logístico repassado ao cliente pela entrega no endereço cadastrado. |

**Especificações Técnicas da Tabela `pedido`:**
* **Chave Primária (PK):** `id_pedido`
* **Chaves Estrangeiras (FKs):** `id_cliente` (REFERENCIA `cliente.id_cliente`). Comportamento referencial padrão (RESTRICT implícito), impedindo a exclusão de um cliente que possua pedidos atrelados.
* **Restrições de Validação de Domínio:** 
  * `CHECK (status IN ('Recebido', 'Em Preparo', 'Saiu para Entrega', 'Finalizado'))` - Impede a inserção de status não mapeados pelo sistema de rastreamento.
  * `CHECK (forma_pagamento IN ('Pix na entrega', 'Cartão na entrega'))` - Trava as regras de negócio de checkout *offline* estipuladas para a loja.

#### 4. Tabela: `item_pedido`
Classe associativa que representa o "carrinho" finalizado. Detalha a relação de N:M entre pedidos e produtos, salvaguardando o cenário financeiro do momento da compra.

| Nome da Coluna | Tipo de Dado Físico | Restrições / Chaves | Descrição e Regra de Negócio |
| :--- | :--- | :--- | :--- |
| `id_item` | SERIAL | PK | Identificador único da linha do item dentro de um pedido específico. |
| `id_pedido` | INTEGER | FK, NOT NULL | Vincula a linha de compra ao pedido consolidador pertencente ao cliente. |
| `id_produto` | INTEGER | FK, NOT NULL | Vincula a linha de compra ao cupcake específico do catálogo selecionado pelo cliente. |
| `quantidade` | INTEGER | NOT NULL, CHECK | Número de unidades adquiridas do mesmo cupcake. Valor abatido da `quantidade_estoque` da tabela produto. |
| `preco_unitario` | NUMERIC(10,2) | NOT NULL | Valor do cupcake *no momento exato da compra*. Registrado separadamente para garantir a blindagem financeira contra futuras inflações no cardápio. |
| `subtotal` | NUMERIC(10,2) | NOT NULL | Valor calculado no backend (`quantidade` * `preco_unitario`). Usado para compor o `valor_total` do pedido. |

**Especificações Técnicas da Tabela `item_pedido`:**
* **Chave Primária (PK):** `id_item`
* **Chaves Estrangeiras (FKs):** 
  * `id_pedido` (REFERENCIA `pedido.id_pedido` `ON DELETE CASCADE`). Se um pedido for cancelado e excluído do sistema por um admin, todos os seus itens associados serão automaticamente removidos, prevenindo registros órfãos.
  * `id_produto` (REFERENCIA `produto.id_produto`). Comportamento referencial padrão (RESTRICT), impedindo a exclusão direta (HARD DELETE) de um produto que já foi vendido, forçando o uso do *soft delete* via campo `ativo`.
* **Restrições de Validação de Domínio:** `CHECK (quantidade > 0)` - Bloqueia fraudes ou falhas no frontend onde o usuário tente comprar 0 itens ou quantidades negativas.

### Matriz de Integridade Referencial e Relacionamentos

A arquitetura do banco de dados garante a consistência das regras de negócio através da seguinte cardinalidade:

* **Cliente 1 : N Pedido:** Um único cliente (`cliente.id_cliente`) pode ter realizado dezenas de pedidos ao longo do tempo (Histórico de Compras), porém um `pedido.id_pedido` gerado pertence exclusivamente a um único cliente (1:1 na via contrária).
* **Pedido 1 : N Item_Pedido:** Um pedido consolidado funciona como um "recibo mestre" que pode abrigar diversos itens diferentes (cupcake de morango, cupcake de chocolate, etc.).
* **Produto 1 : N Item_Pedido:** Um produto específico do catálogo de cupcakes pode constar como item em pedidos de milhares de clientes distintos. 

**O Papel Estratégico do *Soft Delete* (Campo `ativo` na tabela `produto`):**
A integridade referencial exige que um registro da tabela `item_pedido` aponte para um ID válido na tabela `produto`. Em um e-commerce, se o administrador excluir definitivamente (via comando `DELETE` SQL) um sabor de cupcake que saiu de linha, o banco de dados dispararia um erro de chave estrangeira, ou, se houvesse um `CASCADE`, apagaria do histórico de vendas todos os recibos contendo aquele sabor. 
A regra de negócio do *Soft Delete* (atualizar o campo `ativo = FALSE`) preserva o histórico financeiro. O produto deixa de ser exibido na Vitrine Virtual do front-end (`SELECT * FROM produto WHERE ativo = TRUE`), mas seu `id_produto`, `nome` e os valores em `item_pedido` continuam intactos no banco. Isso permite a auditoria financeira de relatórios passados e garante que o usuário ainda veja o nome correto do cupcake esgotado na sua tela de "Histórico de Compras".