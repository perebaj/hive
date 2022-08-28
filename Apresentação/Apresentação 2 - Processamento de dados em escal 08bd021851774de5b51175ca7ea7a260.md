# Apresentação 2 - Processamento de dados em escala

<aside>
💡 O objetivo da apresentação é mostrar na prática as funcionalidades do Hive

</aside>

<aside>
💡 Lembrando um pouco sobre o que foi mostrado na primeira apresentação

</aside>

- Data Warehouse

# Hive Introduction

- Artigo (**Hive - a petabyte scale data warehouse using Hadoop - 2010**)

Alguns pontos chave:

- O tamanho dos conjuntos de dados coletados e analisados no setor de inteligência de negócios está crescendo rapidamente, tornando as soluções de armazenamento tradicionais proibitivamente caras.
- A análise escalável em grandes conjuntos de dados tem sido fundamental para as funções de várias equipes do Facebook - tanto de engenharia quanto de não engenharia.
- Toda a infraestrutura de processamento de dados no Facebook antes de 2008 foi usando um RDBMS comercial.

<aside>
💡  Os dados que o Facebook estava gerando, estava crescendo muito rápido - como exemplo, crescemos de um conjunto de dados de 15 TB em 2007 para um conjunto de dados de 700 TB hoje(2010).

</aside>

- Hadoop é uma implementação popular de MapReduce, que possui código aberto e que estava sendo amplamente utilizado em empresas como Yahoo, Facebook.
- No entanto, o modelo de programação MapReduce é de nível muito baixo e exige que os desenvolvedores escrevam programas personalizados que são difíceis de manter e reutilizar.
- **Hive, é uma solução de armazenamento de dados de código aberto construída sobre o Hadoop.**
- O Hive suporta consultas expressas em uma linguagem declarativa como SQL - HiveQL
    - O HiveQL é compilado, por debaixo dos panos, com sintaxes de MapReduce que são executadas usando o Hadoop.
    - O HiveQL permite que os usuários conectem scripts personalizados de MapReduce em consultas SQL
    - A linguagem inclui um **sistema de tipos** com suporte para **tabelas** contendo tipos **primitivos**, coleções como **arrays** e **mapas** e **composições aninhadas** dos mesmos.
    - Hive também inclui um **catálogo do sistema** - **Metastore** - que contém esquemas e estatísticas, úteis na exploração de dados, otimização de consultas e compilação de consultas.

## Motivação:

O fato de o Hadoop já ser um projeto de código aberto que estava sendo usado em escala de petabytes e fornecer escalabilidade usando hardware comum, foi uma proposta muito atraente para o Facebook desenvolver o Hive.

## Seções que vamos abordar

- A seção II descreve o modelo de dados, os sistemas de tipos e o HiveQL. ✅
- A Seção III detalha como os dados nas tabelas do Hive são armazenados no sistema de arquivos distribuído subjacente – HDFS (sistema de arquivos Hadoop). ✅
- A Seção IV descreve a arquitetura do sistema e vários componentes do Hive. ✅

# Sessão 2

<aside>
💡 Palavras reservadas do HiveQL: [https://cwiki.apache.org/confluence/display/hive/languagemanual+ddl](https://cwiki.apache.org/confluence/display/hive/languagemanual+ddl)

</aside>

## 🍜 Data Model and Type System

Semelhante aos bancos de dados tradicionais, o Hive armazena dados em tabelas, onde cada tabela consiste em um número de linhas e cada linha consiste em um número especificado de colunas. Cada coluna tem um tipo associado. O tipo é um **tipo primitivo** ou um **tipo complexo**. Atualmente, os seguintes tipos primitivos são suportados:

- Inteiros – bigint(8 bytes), int(4 bytes), smallint(2 bytes), tinyint(l byte). Todos os tipos inteiros são assinados.
- Números de ponto flutuante – float (precisão simples), double (precisão dupla)
- Corda

O Hive também oferece suporte nativo aos seguintes tipos complexos:

- Arrays associativos – map<key-type, value-type>
- Listas – lista<tipo de elemento>
- Structs – struct<file-name: field-type, … >

### 🙄 Serialização e Desserialização

Em certas situações, as informações de tipo também podem ser fornecidas por um arquivo jar, fornecendo uma implementação correspondente da interface java ObjectInspector e expondo essa implementação através do método getObjectInspector presente na interface SerDe.

Assim é possível importar pacotes de Serialização e Desserialização de dados para criação de tabelas personalizadas.

![Untitled](Apresentac%CC%A7a%CC%83o%202%20-%20Processamento%20de%20dados%20em%20escal%2008bd021851774de5b51175ca7ea7a260/Untitled.png)

## Query Language

Recursos SQL tradicionais como subconsultas de cláusulas, vários tipos de junções - junções internas, externas esquerdas, junções externas e externas direitas, produtos cartesianos, agrupar bys e agregações, união de todos, criar tabela como seleção e muitas funções úteis em tipos primitivos e complexos tornar a linguagem muito parecida com SQL.

### Limitações

Outra limitação está em como as inserções são feitas. Atualmente, o Hive não oferece suporte à inserção em uma tabela ou partição de dados existente e todas as inserções substituem os dados existentes. Assim, tornamos isso explícito em nossa sintaxe da seguinte forma:

Na realidade, essas restrições não são um problema. Raramente vimos um caso em que a consulta não pode ser expressa como um equi-join e, como a maioria dos dados é carregada em nosso warehouse diariamente ou a cada hora, simplesmente carregamos os dados em uma nova partição da tabela para esse dia ou hora.

## MapReduce no SQL 🤮

Além dessas restrições, o HiveQL possui extensões para suportar análises expressas como programas de MapReduce pelos usuários e na linguagem de programação de sua escolha. Isso permite que usuários avançados expressem lógica complexa em programas de redução de MapReduce as consultas HiveQL perfeitamente. Às vezes, essa pode ser a única abordagem razoável, por exemplo. No caso de existirem bibliotecas em python, PHP ou qualquer outra linguagem que o usuário queira utilizar para transformação de dados. O exemplo de contagem de palavras canônicas em uma tabela de documentos pode, por exemplo, ser expresso usando map-reduce da seguinte maneira:

![Untitled](Apresentac%CC%A7a%CC%83o%202%20-%20Processamento%20de%20dados%20em%20escal%2008bd021851774de5b51175ca7ea7a260/Untitled%201.png)

Como mostrado neste exemplo, a cláusula MAP indica como as colunas de entrada (doctext neste caso) podem ser transformadas usando um programa de usuário (neste caso ‘python we_mapper.py’) em colunas de saída (word e cnt). A cláusula CLUSTER BY na subconsulta especifica as colunas de saída, criptografadas para distribuir os dados para os redutores e, finalmente, a cláusula REDUCE especifica o programa do usuário a ser invocado (python wc_reduce.py neste caso) nas colunas de saída do subconsulta.

# Seção 3 - **Data Storage, Serde And File Formats**

## Data Storage

Enquanto as tabelas são unidades de dados lógicos no Hive, os metadados da tabela associam os dados em uma tabela aos diretórios HDFS. As unidades de dados primárias e seus mapeamentos no espaço de nomes HDFS são os seguintes:

- Tabelas _ Uma tabela é armazenada em um diretório em HDFS.
- Partições _ Uma partição da tabela é armazenada em um subdiretório dentro do diretório de uma tabela.
- Buckets _ Um bucket é armazenado em um arquivo dentro do diretório da partição ou da tabela, dependendo se a tabela é particionada ou não.

Uma tabela pode ser particionada ou não particionada. Uma tabela particionada pode ser criada especificando a cláusula PARTITIONED BY na instrução CREATE TABLE conforme mostrado abaixo.

![Untitled](Apresentac%CC%A7a%CC%83o%202%20-%20Processamento%20de%20dados%20em%20escal%2008bd021851774de5b51175ca7ea7a260/Untitled%202.png)

O conceito de unidade de armazenamento final que o Hive usa é o conceito de Baldes. Um bucket é um arquivo dentro do diretório de nível folha de uma tabela ou partição. No momento em que a tabela é criada, o usuário pode especificar o número de buckets necessários e a coluna na qual armazenar os dados. Na implementação atual, essa informação é usada para podar os dados caso o usuário execute a consulta em uma amostra de dados, por exemplo. Uma tabela agrupada em 32 buckets pode gerar rapidamente uma amostra de 1/32, escolhendo examinar o primeiro bucket de dados.

![Untitled](Apresentac%CC%A7a%CC%83o%202%20-%20Processamento%20de%20dados%20em%20escal%2008bd021851774de5b51175ca7ea7a260/Untitled%203.png)

### External Table

Uma tabela externa difere de uma tabela normal apenas porque um comando drop table em uma tabela externa apenas descarta os metadados da tabela e não exclui nenhum dado. Uma queda em uma tabela normal, por outro lado, também descarta os dados associados à tabela.

![Untitled](Apresentac%CC%A7a%CC%83o%202%20-%20Processamento%20de%20dados%20em%20escal%2008bd021851774de5b51175ca7ea7a260/Untitled%204.png)

## Serialization/Deserialization (SerDe)

Como mencionado anteriormente, o Hive pode pegar uma implementação da interface java SerDe fornecida pelo usuário e associá-la a uma tabela ou partição. Como resultado, os formatos de dados personalizados podem ser facilmente interpretados e consultados.

# Seção 4 - **System Architecture And Components**

![Untitled](Apresentac%CC%A7a%CC%83o%202%20-%20Processamento%20de%20dados%20em%20escal%2008bd021851774de5b51175ca7ea7a260/Untitled%205.png)

## Componentes arquiteturais do Hive

Os seguintes componentes são os principais blocos de construção no Hive:

- **Metastore** – O componente que armazena o catálogo do sistema e metadados sobre tabelas, colunas, partições, etc.
- **Driver** – O componente que gerencia o ciclo de vida de uma instrução HiveQL conforme ela se move pelo Hive. O driver também mantém um identificador de sessão e todas as estatísticas de sessão.
- **Query Compiler** – O componente que compila o HiveQL em um gráfico acíclico direcionado de tarefas de mapeamento/redução.
- **Execution Engine** – O componente que executa as tarefas produzidas pelo compilador na ordem de dependência adequada. O mecanismo de execução interage com a instância subjacente do Hadoop.
- **HiveServer** – O componente que fornece uma interface thrift e um servidor JDBC/ODBC e fornece uma maneira de integrar o Hive com outros aplicativos.
- Componentes de clientes como a **interface de linha de comando** (CLI), a interface do usuário da web e o driver JDBC/ODBC.