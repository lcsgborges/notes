# Introdução aos bancos de dados

## O que é um banco de dados

É uma coleção organizada de dados relacionados entre si, armazenados de forma que possam ser consultados, manipulados, protegidos e mantidos com eficiência.

Em outras palavras, um banco de dados é uma forma estruturada de guardar informações para que sistemas, empresas e usuários consigam acessá-las de maneira confiável.

## Dados, informação e conhecimento

### Dado

É um valor bruto, isolado, sem muito significado sozinho.

Exemplos:

- João.
- Brasília.
- 3.500.

### Informação

É o dado colocado em contexto.

Exemplo:

- João mora em Brasília e possui um salário de R$ 3.500,00.

### Conhecimento

É a interpretação da informação para tomar decisões.

Exemplo:

- Clientes de Brasília com renda acima de R$ 3.500,00 podem receber determinada oferta de crédito.

Um banco de dados lida principalmente com o armazenamento e a organização dos **dados**, para que eles possam ser transformados em **informações úteis**.

## Por que usar um banco de dados?

Entre as principais vantagens estão:

- Organização dos dados.
- Redução de redundância.
- Maior segurança.
- Controle de acesso.
- Integridade dos dados.
- Facilidade de consulta.
- Suporte a múltiplos usuários.
- Backup e recuperação.
- Melhor desempenho para grandes volumes de dados.

## Banco de dados e sistema gerenciador de banco de dados (SGBD)

### Banco de dados

É o conjunto de dados armazenados.

Exemplo:

- Clientes.
- Contas.
- Agências.
- Transações.
- Funcionários.

### SGBD — sistema gerenciador de banco de dados

É o software responsável por criar, controlar, acessar, proteger e manipular o banco de dados.

Exemplos:

- PostgreSQL.
- MySQL.
- Oracle Database.
- SQL Server.
- SQLite.
- MongoDB.
- Redis.
- MariaDB.

Uma analogia simples é:

- Banco de dados: os arquivos organizados.
- SGBD: o sistema que gerencia esses arquivos.

## O que o SGBD faz?

Ele fornece uma camada de controle entre os usuários ou as aplicações e os dados armazenados.

Principais funções de um SGBD:

### Armazenamento

O SGBD define como os dados serão armazenados fisicamente em disco ou memória.

### Consulta

Permite recuperar dados de forma eficiente.

Exemplo: buscar todos os clientes de uma determinada cidade.

### Manipulação

Permite inserir, alterar e remover dados.

Exemplos: cadastrar um novo cliente e atualizar o endereço de uma conta.

### Controle de acesso

Define quem pode acessar quais dados.

Exemplos:

- Um atendente pode consultar dados de clientes.
- Um gerente pode aprovar operações.
- Um administrador pode criar usuários.

### Segurança

Protege os dados contra acessos indevidos, falhas ou uso incorreto.

### Integridade

Garante que os dados permaneçam corretos e consistentes.

Exemplo:

- Uma conta bancária não deve estar associada a um cliente inexistente.

### Concorrência

Permite que vários usuários acessem o banco ao mesmo tempo sem corromper os dados.

### Backup e recuperação

Permite recuperar os dados em caso de falhas, apagamentos acidentais ou problemas no sistema.

## Banco de dados baseado em arquivos

Antes dos bancos de dados modernos, muitas aplicações armazenavam informações diretamente em arquivos.

Exemplo:

- `clientes.txt`
- `produtos.csv`

### Problemas do uso direto de arquivos

#### Redundância

O mesmo dado pode aparecer em vários arquivos.

Exemplo:

- O endereço de um cliente aparece no arquivo de pedidos, no arquivo de pagamento e no arquivo de entregas. Se o cliente mudar de endereço, todos os arquivos precisarão ser atualizados.

#### Inconsistência

Quando um dado é atualizado em um arquivo, mas não em outro, surgem informações conflitantes.

Exemplo:

- Em um arquivo, o telefone do cliente é 99999-1111, em outro é 98888-2222.

#### Dificuldade de acesso

Buscar informações específicas pode exigir muito código manual.

#### Falta de segurança

Arquivos simples geralmente não oferecem controle robusto de permissões.

#### Falta de controle de concorrência

Se duas pessoas alteram o mesmo arquivo ao mesmo tempo, pode ocorrer perda ou corrupção de dados.

#### Dificuldade de recuperação

Se o arquivo for apagado ou corrompido, pode ser difícil restaurá-lo corretamente.

## Características de um banco de dados

### Dados relacionados

Os dados não são isolados. Eles possuem relações entre si.

Exemplos:

- Um cliente possui uma ou mais contas.
- Uma conta possui várias transações.
- Uma agência possui vários funcionários.

### Estrutura definida

Os dados seguem alguma forma de organização.

Exemplo:

- Cliente:
  - Nome.
  - CPF.
  - Telefone.
  - Endereço.

### Persistência

Os dados permanecem armazenados mesmo depois que o sistema é encerrado.

### Compartilhamento

Vários usuários e aplicações podem acessar o banco de dados.

### Controle

O acesso aos dados é controlado pelo SGBD.

### Integridade

O banco deve impedir ou reduzir dados incorretos, inválidos ou contraditórios.

## Esquema e instância

### Esquema

O esquema é a estrutura do banco de dados. Ele define como os dados serão organizados.

Exemplo:

- Cliente:
  - ID.
  - Nome.
  - CPF.
  - Telefone.

O esquema é como uma planta ou modelo.

### Instância

A instância é o conjunto de dados armazenados em determinado momento.

Exemplo:

```text
1, Ana Souza, 123.456.789-00, 99999-1111
2, Carlos Lima, 987.654.321-00, 98888-2222
```

A instância muda com frequência, porque dados são inseridos, alterados e removidos.

Resumo:

- Esquema: estrutura.
- Instância: dados atuais.

## Metadados

Metadados são "dados sobre dados". Eles descrevem a estrutura, o tipo, as restrições e outras características dos dados armazenados.

Exemplo:

- Campo: `data_nascimento`.
- Tipo: data.
- Obrigatório: sim.
- Descrição: armazena a data de nascimento do cliente.

O SGBD usa metadados para entender como o banco está organizado.

## Independência dos dados

A **independência de dados** é a capacidade de alterar a estrutura do banco em um nível sem afetar outros níveis do sistema.

### Independência física

Permite alterar a forma como os dados são armazenados fisicamente sem alterar a forma como os usuários ou programas acessam esses dados.

Exemplo:

- Mover os dados para outro disco.
- Alterar a organização interna dos arquivos.

### Independência lógica

Permite alterar a estrutura lógica do banco sem afetar diretamente as aplicações externas.

Exemplos:

- Adicionar um novo campo ao cadastro de clientes.
- Separar um cadastro muito grande em estruturas menores.

## Abstração de dados

Abstração significa esconder detalhes complexos e mostrar apenas o que é necessário.

Em banco de dados, isso é muito importante porque nem todos os usuários precisam saber como os dados são armazenados fisicamente.

### Nível físico ou interno

Descreve como os dados são realmente armazenados.

Exemplo:

- Arquivos, páginas, blocos, índices e estruturas em disco.

Esse nível interessa mais ao SGBD e ao administrador do banco.

### Nível lógico ou conceitual

Descreve quais dados existem e como eles se relacionam.

Exemplo:

- Clientes possuem contas.
- Contas possuem transações.
- Agências possuem funcionários.

Esse nível é importante para projetistas, desenvolvedores e analistas.

### Nível externo ou de visão

Mostra apenas uma parte do banco para determinado usuário ou aplicação.

Exemplo:

- Um atendente vê nome, telefone e agência do cliente.
- O setor financeiro vê saldo, limite e transações.

Esse modelo de níveis ajuda a proteger os dados e simplificar o acesso.

## Modelos de banco de dados

### Modelo hierárquico

Organiza os dados em forma de árvore.

Exemplo:

```text
Banco 
  └── Agência 
          └── Conta 
                 └── Transação
```

Cada registro possui uma relação de pai e filho.

### Modelo em rede

Permite relações mais complexas do que o modelo hierárquico. Um registro pode estar relacionado a vários outros registros.

Exemplo:

- Um cliente pode ter várias contas.
- Uma conta pode ter mais de um cliente.

Foi uma evolução do modelo hierárquico, mas também é mais complexo de gerenciar.

### Modelo relacional

É o modelo mais tradicional e mais usado em sistemas corporativos. Organiza os dados em estruturas chamadas **tabelas**.

Exemplo:

- Tabela `Cliente`.
- Tabela `Conta`.
- Tabela `Agência`.
- Tabela `Transação`.

Cada tabela armazena dados sobre um tipo de entidade.

### Modelo orientado a objetos

Armazena dados na forma de objetos, semelhantes aos usados em linguagens de programação orientadas a objetos.

Exemplo:

- Objeto `Cliente`:
  - `nome`.
  - `cpf`.
  - `contas`.
  - `atualizarEndereco()`.

### Bancos NoSQL

NoSQL é uma família de bancos que não seguem necessariamente o modelo relacional tradicional. Eles surgiram para lidar com cenários como:

- Grande volume de dados.
- Alta escalabilidade.
- Dados sem estrutura fixa.
- Aplicações web distribuídas.
- Leitura e escrita em grande escala.

Principais tipos de bancos NoSQL:

#### Documento

Armazena dados em documentos, geralmente parecidos com JSON.

Exemplo:

```json
{
    "nome": "Ana",
    "cpf": "12345678900",
    "enderecos": [
        {
            "cidade": "Brasília",
            "uf": "DF"
        }
    ]
}
```

Exemplos: **MongoDB** e **CouchDB**.

#### Chave-valor

Armazena dados como pares de chave e valor.

Exemplo:

```text
usuario:1001 -> Ana Souza
sessao:abc123 -> dados da sessão
```

Exemplos: **Redis** e **DynamoDB**.

#### Colunar

Organiza dados por colunas, sendo útil para grandes volumes de dados distribuídos.

Exemplos: **Cassandra** e **HBase**.

#### Grafos

É focado em representar relacionamentos complexos.

Exemplo:

```text
Pessoa A conhece Pessoa B.
Pessoa B trabalha na Empresa X.
Pessoa A comprou Produto Y.
```

Exemplos: **Neo4j** e **Amazon Neptune**.

## Entidades, atributos e relacionamentos

Mesmo antes de estudar SQL, precisamos entender três conceitos fundamentais da modelagem:

### Entidade

É algo do mundo real sobre o qual queremos armazenar dados.

Exemplos:

- Clientes.
- Contas.
- Produtos.
- Funcionários.

### Atributo

É uma característica de uma entidade.

Exemplo:

- Cliente:
  - Nome.
  - CPF.
  - Telefone.
  - Endereço.

### Relacionamento

É a associação entre entidades.

Exemplo:

- Cliente possui Conta.
- Pedido contém Produto.
- Funcionário trabalha em Departamento.

## Integridade dos dados

Integridade significa manter os dados corretos, válidos e coerentes.

Um banco de dados deve evitar situações como:

- Conta bancária sem cliente.
- Transação com valor inválido.
- CPF duplicado para dois clientes diferentes.

### Integridade de entidade

Cada registro deve ser identificável de forma única.

Exemplo:

- Cada cliente deve ter um identificador único.

### Integridade referencial

Relacionamentos entre dados devem ser válidos.

Exemplo:

- Uma conta não pode estar associada a um cliente inexistente.

### Integridade de domínio

Os valores devem respeitar um conjunto permitido.

Exemplo:

- Idade não pode ser negativa.
- Data de nascimento não pode ser uma data futura.
- Saldo não deveria aceitar texto no lugar de número.

### Integridade de negócio

Regras específicas do domínio da aplicação.

Exemplo:

- Uma transferência bancária só pode ocorrer se houver saldo suficiente.

## Redundância e inconsistência

### Redundância

Acontece quando a mesma informação é armazenada repetidamente sem necessidade.

Exemplo:

- O endereço do cliente aparece em vários lugares diferentes.

Nem toda redundância é proibida. Às vezes, ela pode ser usada por motivos de desempenho.

### Inconsistência

Acontece quando os dados entram em conflito.

Exemplo:

- Em um cadastro, o cliente mora em Brasília.
- Em outro cadastro, o mesmo cliente mora em Goiânia.

Uma das funções do projeto de banco de dados é reduzir as redundâncias desnecessárias e evitar inconsistências.

## Transações

Uma transação é uma sequência de operações que deve ser tratada como uma única unidade lógica de trabalho.

Exemplo: transferência bancária.

1. Tirar R$ 100,00 da Conta A.
2. Adicionar R$ 100,00 na Conta B.

Não pode acontecer de o dinheiro sair da Conta A e não entrar na Conta B.

Uma transação deve seguir propriedades conhecidas como **ACID**.

### Atomicidade

A transação acontece por completo ou não acontece.

> Ou a transferência inteira é feita, ou nada é feito.

### Consistência

A transação leva o banco de um estado válido para outro estado válido.

> Depois da transferência, os saldos precisam continuar corretos.

### Isolamento

Transações simultâneas não devem interferir incorretamente umas nas outras.

> Dois saques ao mesmo tempo não podem gerar um saldo incorreto.

### Durabilidade

Depois que a transação é confirmada, seus efeitos devem permanecer, mesmo em caso de falha posterior.

> Se o sistema cair depois da confirmação, a transferência não deve desaparecer.

## Concorrência

Concorrência ocorre quando vários usuários ou sistemas acessam o banco de dados ao mesmo tempo.

Exemplo:

- Dois caixas tentando atualizar a mesma conta.
- Vários clientes comprando o último item em estoque.
- Muitos usuários acessando um sistema bancário simultaneamente.

Sem controle adequado, podem ocorrer problemas como:

- Perda de atualização.
- Leitura de dados incorretos.
- Conflitos entre operações.
- Inconsistência de saldo ou estoque.

O SGBD possui mecanismos para controlar a concorrência, garantindo que múltiplas operações possam ocorrer de forma segura.

## Segurança em banco de dados

Um banco de dados geralmente armazena informações sensíveis.

Exemplos:

- CPF.
- Senhas.
- Dados bancários.
- Endereços.
- Histórico financeiro.

### Autenticação

Verifica quem é o usuário.

Exemplo:

- Login e senha.
- Certificado digital.

### Autorização

Define o que o usuário pode fazer.

Exemplo:

- Consultar dados.
- Inserir registros.
- Alterar informações.
- Excluir registros.

### Criptografia

Protege dados para que não sejam lidos facilmente por pessoas não autorizadas.

### Auditoria

Registra ações realizadas no banco.

Exemplo:

- Quem acessou?
- Quando acessou?
- Qual dado foi alterado?
- Qual era o valor anterior?

## Backup e recuperação

Backup é a cópia de segurança dos dados.

### Backup completo

Copia todos os dados.

### Backup incremental

Copia apenas o que mudou desde o último backup.

### Backup diferencial

Copia o que mudou desde o último backup completo.

A recuperação é o processo de restaurar os dados a partir de um backup ou de registros mantidos pelo SGBD.

Em sistemas críticos, como bancos, hospitais e órgãos públicos, backups e recuperações são fundamentais.

## Usuários de um banco de dados

### Administrador de banco de dados (DBA)

Responsável por gerenciar o banco.

Atividades comuns:

- Criar estruturas.
- Gerenciar usuários.
- Controlar permissões.
- Monitorar o desempenho.

### Projetista de banco de dados

Responsável por modelar a estrutura do banco.

Define entidades, atributos, relacionamentos e restrições.

### Desenvolvedor

Cria aplicações que acessam o banco de dados.

### Usuário final

Utiliza o sistema sem necessariamente conhecer o banco por trás.
