# Busca Híbrida

O termo "busca híbrida" se refere à combinar diferentes resultados de diferentes sistemas de busca (mas que retornam os
mesmos documentos). Tipicamente dois são os sistemas de busca, um baseado em embedding e outro em full text search
tradicional, um casando semântica e o outro sintaxe, respectivamente.

## Busca híbrida com Redis

O módulo RediSearch permite realizar tanto a busca quanto a indexação full text, embeddings, ou ambos.

No caso da busca híbrida, primeiro é realizado casamento via full text e posteriormente é utilizado os embeddings para
ranquear os resultados. Isso contrasta com o que é tipicamente feito com outros motores de busca em que as buscas são
realizadas separadamentes e depois combinadas.

Infelizmente não existe funcionalidade pronta para alterar este comportamento, se for desejável combinar os resultados
de ambas buscas isoladas é necessário realizar duas buscas no Redis e implementar a combinação dos resultados.

## Carregando o Redis a partir de uma base sbretrieval

Para executar o exemplo é necessário utilizar um Redis com o módulo RediSearch instalado (v2.10.15 foi utilizado). Por
conveniência, pode-se utilizar a versão Redis Stack que já possui o módulo instalado e configurado (v7.4.0 foi
utilizado).

O script abaixo carrega o Redis a partir de uma base Postgres com o schema da biblioteca [sbretrieval](https://github.com/Framework-System/summary-based-retrieval).
Basta alterar as variáveis no início do script (como `pg_connection_string`) e executá-lo sem nenhum parâmetro.

```py
#!/usr/bin/env python3

from asyncio import run

from asyncpg import connect # pip install asyncpg
from numpy import fromstring, float32 # pip install numpy
from redis import Redis # pip install redis


async def main():

    # configurable values used on the script
    pg_connection_string = "postgresql://code_indexer:code_indexer@localhost:5432/code_index"
    index_name = "my_index"
    embedding_dimension = 768
    key_prefix = "document:"

    # connecting and retrieving records from postgres
    print("Conectando ao postgres...")
    pg_connection = await connect(pg_connection_string)

    print("Buscando registros das tabelas de embeddings e sumários...")
    records = await pg_connection.fetch("""
        SELECT e.document_key, s.summary, e.embedding::text AS embedding_text
        FROM public.sbretrieval_embeddings e
        JOIN public.sbretrieval_summaries s USING (document_key)
    """)

    # connecting to redis and creating the index
    print("Conectando ao redis...")
    redis_connection = Redis(decode_responses=False)

    print("Criando índice no redis...")
    redis_connection.execute_command(
        # creating and index that is on hashes
        "FT.CREATE", index_name, "ON", "HASH",

        # all keys starting with this prefix will be considered part of the index
        "PREFIX", "1", key_prefix,

        # the schema goes below this
        "SCHEMA",

        # field indexed for full text search
        "summary", "TEXT",

        # field indexed for vector similarity search with HNWS
        # 6 is the count for the rest of the parameters below
        "embedding", "VECTOR", "HNSW", "6",

        # the numerical datatype of the embedding values
        "TYPE", "FLOAT32",

        # the dimensions of the embedding (its length)
        "DIM", embedding_dimension,

        # using cosine distance for vector comparison
        "DISTANCE_METRIC", "COSINE"
    )

    # inserting the records into redis
    print(f"Inserindo {len(records)} registros no redis...")
    for record in records:

        # key name
        document = record["document_key"]
        key = f"{key_prefix}{document}"

        # key value
        vector_bytes = fromstring(record['embedding_text'].strip("[]()"), sep=",", dtype=float32).tobytes()
        value = {"summary": record['summary'], "embedding": vector_bytes}

        redis_connection.hset(key, mapping=value)

run(main())
```

Exemplo de saída do script:

```
Conectando ao postgres...
Buscando registros das tabelas de embeddings e sumários...
Conectando ao redis...
Criando índice no redis...
Inserindo 500 registros no redis...
```

## Buscando no Redis

Com o Redis carregado, basta executar o seguinte script para realizar a busca  Logo no ínicio do script você encontra as
variáveis que influenciarão o resultado do busca (como `query` e `limit`).

```py
#!/usr/bin/env python3

from google.generativeai import embed_content # pip install google-generativeai
from numpy import array, float32  # pip install numpy
from redis import Redis  # pip install redis

# configurable values used on the script
query = 'apropriação de horas'
limit = 5
index_name = "my_index"

# connecting to a local redis
redis_connection = Redis()

# generating embedding with gemini
response = embed_content("models/text-embedding-004", query)
embedding =  array(response["embedding"], dtype=float32).tobytes()

# executing the search on redis
results = redis_connection.execute_command(
    # running a search using RediSearch module ("FT" stands for "full text") on our index
    "FT.SEARCH", index_name,

    # the following: (text filter) => [vector filter]
    # means: 'find matching documents by text filter first and then apply the k nearest neighbors'
    # note that this is not combining two individual searches, one is actually limiting the other
    f"(@summary:{query}) => [KNN {limit} @embedding $my_embedding AS distance]",

    # sorting by the distance of this search
    "SORTBY", "distance",

    # telling how many parameters are we passing (the next two below, for the embedding)
    "PARAMS", "2",

    # passing our embedding as parameter to the query
    "my_embedding", embedding,

    # the type of dialect to be used by RediSearch, the only option non-deprecated at time of writing
    # https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/dialects/#dialect-2
    "DIALECT", "2",
)

# the first value of results is an integer with the count of retrieved records
print(f"Foram encontrados {results[0]} resultados para a busca \"{query}\"")
print()

# the rest of the values are pairs of key (which is the document path) and value (the indexed hash)
for i in range(1, len(results), 2):

    # getting the key as a string, which is the document path
    document = results[i].decode()

    hash_fields = results[i + 1]

    # fields[0] is the string "distance" and fields[1] its value
    distance = hash_fields[1].decode()

    # fields[2] is the string "summary" and fields[3] its value
    summary = hash_fields[3].decode()[:50]

    print(f"Documento: {document}")
    print(f"Distância: {distance}")
    print(f"Sumário: {summary}...")
    print()
```

Exemplo de saída do script:

```
Foram encontrados 5 resultados para a busca "apropriação de horas"

Documento: document:frwk-apropriacao-web/src/app/pages/appropriation/components/appropriation-historic/appropriation-historic.component.html
Distância: 0.322598397732
Sumário: A página permite gerar relatórios de apropriações ...

Documento: document:frwk-apropriacao-web/src/app/pages/appropriation/components/appropriation-new/appropriation-new.component.html
Distância: 0.342845082283
Sumário: O código apresenta um formulário para apropriação ...

Documento: document:frwk-apropriacao-alerts-service/src/main/java/br/com/frwkapp/model/service/AppropriationService.java
Distância: 0.365364074707
Sumário: A classe `AppropriationService` é um serviço Sprin...

Documento: document:frwk-apropriacao-web/src/app/pages/apropriacao/apropriacao.component.html
Distância: 0.381155252457
Sumário: O código HTML e Angular representa uma interface d...

Documento: document:frwk-apropriacao-api-tests/src/test/java/Tests/AppropriationTest.java
Distância: 0.386589705944
Sumário: A classe `AppropriationTest` testa a API de apropr...
```

## Casamento de texto no Postgres

Antes de demonstrarmos como adaptar a base sbretrieval para casamento textual (sem embeddings), alguns conceitos
básicos:

- `tsvector`: Lista ordenada de lexemas. Lexema é a raiz da palavra ("cant" é o lexema da palavra "cantando" por
  exemplo).
- `to_tsvector()`: Função que converte uma string para `tsvector`.
- `tsquery`: Uma busca textual composta por palavras e operadores `&` (AND), `|` (OR), e `!` (NOT).
- `to_tsquery()`: Converte uma string (em formato de query) para um `tsquery`.
- `plainto_tsquery()`: Quebra uma string em palavras individuais e converte em um `tsquery` (operação AND).
- `@@`: Operador de casamento de texto, espera um `tsvector` na esquerda e um `tsquery` na direita.

Exemplo de conversão de string para `tsvector`:

```
=# SELECT to_tsvector('portuguese', 'um pequeno jabuti xereta viu dez cegonhas felizes');
                             to_tsvector
---------------------------------------------------------------------
 'cegonh':7 'dez':6 'feliz':8 'jabut':3 'pequen':2 'viu':5 'xeret':4
```

Exemplo de conversão de string para `tsquery`:

```
=# SELECT to_tsquery('portuguese', 'tartaruga | jabuti');
      to_tsquery
----------------------
 'tartarug' | 'jabut'
```


Exemplo de casamento de texto com `@@`:

```
=# SELECT to_tsvector('portuguese', 'um pequeno jabuti xereta viu dez cegonhas felizes') @@ to_tsquery('portuguese', 'jabuti');
 ?column?
----------
 t
```

## Adicionando suporte a busca textual no banco sbretrieval

De maneira conveniente, podemos adicionar uma nova coluna do tipo `tsvector` e configurar a tabela para gerar o vetor
automaticamente (`GENERATED`) além de armazenar esse valor gerado (`STORED`):

```sql
ALTER TABLE sbretrieval_summaries 
ADD COLUMN summary_tsv tsvector 
GENERATED ALWAYS AS (to_tsvector('portuguese', summary)) STORED;
```

Exemplo utilizando a nova coluna, buscando os top 10 resultados com query "apropriação de horas" (operação AND):

```
=# SELECT s.document_key, ts_rank_cd(s.summary_tsv, q) AS rank
FROM   sbretrieval_summaries AS s,
       plainto_tsquery('portuguese', 'apropriação de horas') AS q
WHERE  s.summary_tsv @@ q
ORDER  BY rank DESC
LIMIT  10;
                                                document_key                                                 |    rank    
-------------------------------------------------------------------------------------------------------------+------------
 frwk-apropriacao-alerts-service/src/main/java/br/com/frwkapp/model/service/AppropriationService.java        | 0.15022708
 frwk-apropriacao-app-mobile/lib/src/screens/home/home_tab_screen.dart                                       |       0.14
 frwk-apropriacao-app-service/src/main/java/br/com/frwkapp/model/service/parameters/ParametersBase.java      | 0.13333334
 frwk-apropriacao-alerts-service/src/main/java/br/com/frwkapp/model/service/parameters/ParametersBase.java   | 0.13333334
 frwk-apropriacao-reports-service/src/main/java/br/com/frwkapp/model/service/AppropriationService.java       | 0.12222222
 frwk-apropriacao-web/src/app/pages/relatorio/rel-apropriacao/rel-apropriacao.component.html                 | 0.11325758
 frwk-apropriacao-web/src/app/pages/apropriacao/apropriacao.component.html                                   |  0.1090397
 frwk-apropriacao-alerts-service/src/main/java/br/com/frwkapp/model/repository/AppropriationRepository.java  |    0.10625
 frwk-apropriacao-reports-service/src/main/java/br/com/frwkapp/model/repository/AppropriationRepository.java |    0.10625
 frwk-apropriacao-web/src/app/pages/home/home.component.html                                                 | 0.10277778
(10 rows)
```

## Combinando resultados com Reciprocal Rank Fusion (RRF)

A fórmula mais utilizada para combinar os resultados é a Reciprocal Rank Fusion (RRF):

```math
\text{RRF}(d) = \sum \frac{1}{k + \text{rank}_i(d)}
```

- `d`: documento da base de dados.
- `k`: constante para reduzir o impacto de outliers (geralmente `60`).
- `rankᵢ(d)`: rank do documento `d` na lista de resultados `i`.

Utilizando a fórmula com apenas dois sistemas de busca e `k = 60`:

```math
\text{RRF}(d) = \frac{1}{k + \text{rank}_1(d)} + \frac{1}{k + \text{rank}_2(d)}
```

Ou seja, novos scores são computados com a fórmula e, uma vez ordenados, tem-se uma única lista de resultados que
representa a combinação de mais de uma lista.

## Referências

- [Doing RAG? Vector search is \*not\* enough](https://techcommunity.microsoft.com/blog/azuredevcommunityblog/doing-rag-vector-search-is-not-enough/4161073)
- [FT.CREATE](https://redis.io/docs/latest/commands/ft.create/)
- [FT.SEARCH](https://redis.io/docs/latest/commands/ft.search/)
- [Chapter 12. Full Text Search](https://www.postgresql.org/docs/current/textsearch.html)
- [Relevance scoring in hybrid search using Reciprocal Rank Fusion (RRF)](https://learn.microsoft.com/azure/search/hybrid-search-ranking)
