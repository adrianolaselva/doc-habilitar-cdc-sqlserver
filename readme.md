
# Habilitar change data capture no SqlServer.

## Criação de banco de dados

```sql
create database example_cdc
```

## Criação tabela para testes

```sql
create table users(
	id integer identity,
	name varchar(90) not null,
	last_name varchar(90) not null,
	email varchar(90) not null,
	unique(email),
	primary key(id)
)
```

## Habilitar CDC para o banco de dados

```sql
USE example_cdc
EXEC sys.sp_cdc_enable_db  
```

## Desabilitar CDC para o banco de dados

```sql
USE example_cdc
EXEC sys.sp_cdc_disable_db
```

## Habilitar CDC para uma tabela do banco de dados

```sql
USE example_cdc
EXEC sys.sp_cdc_enable_table  
@source_schema = N'dbo',  
@source_name   = N'users',  
@role_name     = NULL,
@supports_net_changes = 1  
```

## Desabilitar CDC para uma tabela do banco de dados

```sql
USE example_cdc
EXEC sys.sp_cdc_disable_table  
@source_schema = N'dbo',  
@source_name   = N'users',
@capture_instance = N'dbo_users'
```

## Validar se o CDC esta habilitado para o banco de dados

```sql
select is_cdc_enabled,* from sys.databases where name in ('example_cdc')
```

![Resultado da consulta de verificação de que se o cdc se encontra habilitado para a base de dados](images/example_cdc-verify-database-is-enable.png)

## Configuração de tempo em que um log fica armazenado

No CDC, há um processo de limpeza automática que é executado em intervalos regulares. Por padrão, o intervalo é de 3 dias, mas pode ser configurado. Observamos que, quando ativamos o CDC no banco de dados, existe um procedimento armazenado do sistema adicional criado com o nome sys.sp_cdc_cleanup_change_table, que limpa todos os dados rastreados no intervalo. 

```sql
USE example_cdc
EXEC sys.sp_cdc_cleanup_change_table   
  @capture_instance='dbo_users',   
  @low_water_mark=NULL,  
  @threshold=5000
```

## Functions de consultas de alterações

```sql
DECLARE @begin_time DATETIME;
DECLARE @end_time DATETIME;
DECLARE @begin_lsn BINARY(10); 
DECLARE @end_lsn BINARY(10);

SELECT @begin_time = GETDATE()-1, @end_time = GETDATE();
SELECT @begin_lsn = sys.fn_cdc_map_time_to_lsn('smallest greater than', @begin_time); 
SELECT @end_lsn = sys.fn_cdc_map_time_to_lsn('largest less than or equal', @end_time);

SELECT * FROM example_cdc.cdc.fn_cdc_get_all_changes_dbo_users(@begin_lsn,@end_lsn,'all') 
```

![Listagem de evento gerado](images/example_cdc-log-1.png)


## Schema gerado para armazenar os logs de alterações na base de dados

![Estrutura de dados após habilitado CDC](images/example_cdc-after-enable.png)

## Functions, procedures e tabelas geradas por schema quando habilitado o CDC

![Functions, procedures e tabelas geradas por schema quando habilitado o CDC](images/example_cdc-fns-per-table.png)

## Exemplo de function gerada para obter os dados atravéz do CDC

![Exemplo de function gerada para obter os dados atravéz do CDC](images/example_cdc-fn-per-table.png)

## Fluxo de dados do Change Data Capture

A ilustração seguinte mostra o fluxo de dados principal para a captura de dados de alteração.

![Fluxo de dados do Change Data Capture](images/example_cdc-workflow-cdc.png)


## Resalvas

- No caso do Sqlserver as informações ficam armazenadas não utilizando logs a nível de arquivo, e sim a nível de tabelas, neste caso, sendo assim deve ser utilizado com cautela, pois se o banco de dados já se encontra com baixa performance habilitando o cdc você pode estar gerando outros problemas maiores.

![Arquitetura da solução](images/example_cdc-resalvas.png)

## Alternativa para não honerar a base de dados relacional

Uma alternativa a não ser discartada é fazer uso de persistência poliglota, ou seja, os dados continuam sendo persistidos da mesma forma e uma biblioteca pode ser adicionada para fazer este controle, assim não honerando o banco de dados relacional e possibilitando o uso de bancos multi-paradigma.

Segue algumas referências:

- [laravel-scout](https://laravel.com/docs/5.8/scout)
- [bonsai-elasticsearch-rails](https://docs.bonsai.io/article/97-ruby-on-rails)
- [Como manter o Elasticsearch sincronizado com um banco de dados relacional usando o Logstash](https://www.elastic.co/pt/blog/how-to-keep-elasticsearch-synchronized-with-a-relational-database-using-logstash)

## Conclusão

Na minha humilde opinião o CDC deve ser utilizado com parcimônia, pois se a base de dados já possui problemas de performance, o mesmo tende aumentar, dado que os dados são armazenados utilizando do próprio SGBC, e o volume de escritas aumenta de forma exponencial. Ou Seja, em uma base de dados que já possui problemas de performance, muitas vezes pelo excesso de normalização dos dados esta abordagem deve ser aplicada com extrema cautela. Outro ponto é que por um banco relacional apenas escala de forma vertical, dado este cenário, o preço e a mão de obra de infra-estrutura tente a aumentar até chegar ao limite da tecnologia (limite de hardware) e tornar inviável manter a infra-estrutura.

## Referências


- [Habilitar e desabilitar o Change Data Capture (SQL Server)](https://docs.microsoft.com/pt-br/sql/relational-databases/track-changes/enable-and-disable-change-data-capture-sql-server?view=sql-server-2017)

- [Sobre o change data capture (SQL Server)](https://docs.microsoft.com/pt-br/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-2017)

- [Tabelas Change Data Capture (Transact-SQL)](https://docs.microsoft.com/pt-br/sql/relational-databases/system-tables/change-data-capture-tables-transact-sql?view=sql-server-2017)

- [Procedimentos armazenados de captura de dados de alteração (Transact-SQL)](https://docs.microsoft.com/pt-br/sql/relational-databases/system-stored-procedures/change-data-capture-stored-procedures-transact-sql?view=sql-server-2017)
