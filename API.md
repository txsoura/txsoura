# Estrutura de API para um CRUDL simples



## CRUDL

Abaixo segue uma tabela com a estrutura de apis para um CRUDL (create read update delete list)

| METHOD    | URI                   | NAME              |
| --------- | --------------------- | ----------------- |
| GET HEAD  | api/v1/exchanges      | exchanges.index   |
| POST      | api/v1/exchanges      | exchanges.store   |
| GET HEAD  | api/v1/exchanges/{id} | exchanges.show    |
| PUT PATCH | api/v1/exchanges/{id} | exchanges.update  |
| DELETE    | api/v1/exchanges/{id} | exchanges.destroy |



## SoftDelet

Em alguns casos, geralmente quando o model em questão faz parte da funcionalidade principal do sistema, não queremos perder o rastro de atividades do modele então preferimos trabalhar com **SOFT DELETE**, que consistem em ativar ou desativar o model. Esta prática é muito comum no caastro de clientes ou produtos, nesse caso além dos métodos tradicionais outros dois devem ser criados, são eles:

| METHOD | URI                           | NAME              |
| ------ | ----------------------------- | ----------------- |
| PUT    | api/v1/exchanges/{id}/disable | exchanges.disable |
| PUT    | api/v1/exchanges/{id}/restore | exchanges.restore |



Observação: Nas apis que ustilizam SOFT DELETE por padrão as listagens devem trazer apenas models ativos.

No caso das APIS que optem por utilizar SOFT DELETE a api por padrão listará apenas models ativose por isso dois filtros importantes que devem ser criados são:

- `?onlyTrashed=1`: retorna apenas models que foram desativados
- `?withTrashed=1`: retorna models de todos os status

# Parâmentros Default

## Exemplo de model

```typescript
interface Post{
   id: number;
   title: string; 
   conten: string;
   rating: number; 
   updated_at:string
   created_at:string;    
   deleted_at:string; // apenas quando com SoftDelet
}
```

## Query serach

Exemplo: `?q=a`

Farão consulta com operador *like* em várias colunas da entidade, no caso a cima, iria gerar a seguinte query no sql:

```sql
select * from title like “%a%” OR content like “%a%”
```



## Conditional query

Exemplo: `?title=my title&rating=3` 

Na maioria das vezes as os nomes dos atributos do model podem ser utilizados como regra condicional. (perceba o AND aqui):

```sql
select * from title = “my title” AND rating = 3”
```



## Where



Exemplo: `?categoria_id=1` 

Para filtrar a lista por outros models relacionados, geralmente representados por model_id ou models_ids, por exemplo para filtrar uma lista e models por categoria.

Vale lembrar que sempre que se o parâmetro em questão estiver no singular, deveráreceber apenas um id, e quando no plural receberá um array de ids.

```sql
select * from model where categoria_id = 3
```



## Sort 

Exemplo: `?sort=-rating,created_at`

Ordena os resultados. Para utilizar ordenação decrescente utilizamos o sinal de `-`na frente do atributo, e para crescente basta remover o sinal da frente. Além disso é possível utilizar mais de um parâmetro de ordenação separando os mesmo por vírgula.

```sql
select * from order by rating DESC, created_at ASC”
```

# Filter by date

Exemplo: `?dt_column=created_at&dt_start=2018-05-05&dt_end=2019-05-05` 

Filtra os dados  por intervalos de datas, nesse caso estou pegando todos os registros no período de 2018-05-05 á 2019-05-05 utilizando como parâmetro a coluna `created_at` que representa a data de criação . 

Pode-se ainda especificar a hora.
Formatos válidos:  `Y-m-d` ou  `Y-m-d H:i` ou `Y-m-d H:i:s`

Destes 3 parametros o unico que é obrigatório é `dt_column`:

- quando passar apenas `dt_start` ira trazer registros apatir desta data pra frente. 
- quando passar apenas `dt_end` ira trazer registros menores ou igual a data . 

```sql
select * from created_at >= "2018-05-05" AND created_at <= "2019-05-05”
```

# Filter by value

Exemplo: `?vl_column=rating&vl_min=2&vl_max=4` 

De forma semelhante ao filtro por data primeiramente é necessaŕia especificar qual coluna deverá ser consultada através do  `vl_column`e na sequência informa o intervalo, nesse caso estou pegando todos os registros com rating entre 2 e 4. 
Formatos válidos:  float, int

Destes 3 parâmetros o único que é obrigatório é `vl_column`:

- quando passar apenas `vl_min` ira trazer registros maior ou igual. 
- quando passar apenas `vl_max` ira trazer registros menores ou igual. 

```sql
select * from rating >= 2 AND rating <= 4
```

# Limit

Exemplo: `?take=5`

Este parâmetro determina a quantidade máxima de resultados que quer para sua pesquiza, no caso do exemplo quero 5 resultados. Este parâmetro pode ser combinado os demais para determinar a quantidade de resultados. Para pegar todos os resultados disponíveis basta passar  `?take=-1`

```sql
select * from limit 5
```

# Paginate

Exemplo: `?paginate=5&page=1`

Configura a paginação dos resultados sendo o parâmetro  `paginate` resposável por determinar a quantidade de resultados por página e `page` por determinar a página em questão que quer. 

No caso do exemplo acima deverá paginar de 5 em 5 resultados começando da pagina 1.
O paramentro `page`é opcional.

Nesse caso o resposta do servidor será um pouco diferente para permitir que seja construído um objeto de paginação, veja:

### Exemplo interface 

```typescript
export interface Links {
    first: string;
    last: string;
    prev?: any;
    next: string;
}

export interface Meta {
    current_page: number;
    from: number;
    last_page: number;
    path: string;
    per_page: string;
    to: number;
    total: number;
}

export interface ResponsePaginate<T> {
    data: T[];
    links: Links;
    meta: Meta;
    includes: string[];
}

```



### Exemplo JSON:

```json
{
    "data": [...],
    "links": {
        "first": "http://services.sifra3.local/api/v1/crm/campanhas?paginate=5&page=1",
        "last": "http://services.sifra3.local/api/v1/crm/campanhas?paginate=5&page=2",
        "prev": null,
        "next": "http://services.sifra3.local/api/v1/crm/campanhas?paginate=5&page=2"
    },
    "meta": {
        "current_page": 1,
        "from": 1,
        "last_page": 2,
        "path": "http://services.sifra3.local/api/v1/crm/campanhas",
        "per_page": "5",
        "to": 5,
        "total": 10
    },
    "includes": [
        "operadores"
    ]
}

```

# Includes 

Exemplo: `?include=tags`

Sempre que passar include, será incluso no obejto da resposta o tipo solicitado,  pode-se usar encademaneto, por exemplo:

`include=authors.city.state.country`  neste caso sera adicionado os autores e tem o objeto cidade com stado dentro com outro obejto pais aninhado.

Pode ser combinado mais de um include usando a virgula (,) para separar, por exemplo: `include=authors.city.state.country,tags`   neste caso além do objeto autores também teremos um array de tags.

Vale lembrar que sempre que o include estiver no singular, sera um objeto simples, quando no plural será passado um array.

> Observação: os includes dentro do objetos são puros, ou seja, eles não sofrem com os parâmetros da URL, tais como: q, sort, paginate, take...  API sempre vai trazer todos os relacionados na integra.

## Filter 

Example: `?filter=bestRatingOfWeek`

Esses filtros são muito particular de cada projeto, e o mesmo tem que ser solicitado a criação para o dev backend, mais no geral toda vez que se precisa de uma consulta mais avançada pode ser usando esse apelo de filter, também pode ser passado mais de um filtro usando o separado pipe (|) 

Nesse exemplo a cima  é provavel que eu faria uma consulta nos posts com rating 4 e 5 rankeado na última semana.  Sempre que tiver um filtro disponivel tentar colocar no cabeçalho do postman.

## API Responses

### 40x


| NOME                          | CÓDIGO | DESCRIÇÃO                                                    |
| ----------------------------- | ------ | ------------------------------------------------------------ |
| HTTP_BAD_REQUEST              | 400    | erro de validação                                            |
| HTTP_UNAUTHORIZED             | 401    | (não logado / sem token, token expirado ou invalido          |
| HTTP_PAYMENT_REQUIRED         | 402    | falta de pagamento                                           |
| HTTP_FORBIDDEN                | 403    | (user logado mais sem permissão de acesso, geralmente por conta de regras de ACL ) |
| HTTP_NOT_FOUND                | 404    | ( URL nao existe ou /:id nao ixiste, mas quando for assim retorno erro_code => model_not_found) |
| HTTP_METHOD_NOT_ALLOWED       | 405    | ( url existe, mas nao verbo que vc esta passando, explo vc da passando POST mais a rota só suporta GET ) |
| HTTP_REQUEST_ENTITY_TOO_LARGE | 413    | quando servidor nao suportar o tamanho do UPLOAD             |
| HTTP_TOO_MANY_REQUESTS        | 429    | muitas requests em um intervalo X de tempo                   |
| HTTP_INTERNAL_SERVER_ERROR    | 500    | Erro interno                                                 |
| HTTP_NOT_IMPLEMENTED          | 501    | ainda em desenvolvimento                                     |
| HTTP_BAD_GATEWAY              | 502    | servidor quando tem agum erro, geralmente quando deu limit de espaço, ram, nginx fora... |
| HTTP_SERVICE_UNAVAILABLE      | 503    | webservice foi desligado (php artisan down )                 |
| HTTP_GATEWAY_TIMEOUT          | 504    | demorou mais do que o time limite de execução configurado    |


### 20x
| NOME         | CÓDIGO | DESCRIÇÃO                             |
| ------------ | ------ | ------------------------------------- |
| HTTP_OK      | 200    | resposta OK ou atualizado com sucesso |
| HTTP_CREATED | 201    | criado com sucesso                    |





