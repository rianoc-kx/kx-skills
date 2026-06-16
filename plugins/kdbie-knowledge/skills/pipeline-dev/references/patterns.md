# Common Pipeline Patterns

Reference patterns for SP pipelines. Adapt to the user's actual topic names, schemas, and config —
don't copy blindly.

---

## Kafka → decode → stream

```q
.qsp.run
    .qsp.read.fromKafka[`trades; ("kafka:9092")]
    .qsp.decode.json[]
    .qsp.map[{ update newcol: price * size from x }]
    .qsp.write.toStream[]
```

```python
sp.run(
    sp.read.from_kafka('trades', 'kafka:9092')
    | sp.decode.json()
    | sp.map(lambda x: x.assign(newcol=x['price'] * x['size']))
    | sp.write.to_stream()
)
```

---

## S3 → CSV → database (ETL)

```q
.qsp.run
    .qsp.v2.read.fromAmazonS3["s3://bucket/data.csv"; "eu-west-1"]
    .qsp.decode.csv[schema; .qsp.use enlist[`exclude]!enlist excludedCols]
    .qsp.transform.renameColumns[`old!`new]
    .qsp.v2.write.toDatabase[`mytable; .qsp.use `directWrite`database!(1b; "mydb")]
```

```python
sp.run(
    sp.read.from_amazon_s3('s3://bucket/data.csv', 'eu-west-1', api_version=2)
    | sp.decode.csv(schema, exclude=excluded_cols)
    | sp.transform.rename_columns({'old': 'new'})
    | sp.write.to_database('mytable', direct_write=True, database='mydb', api_version=2)
)
```

---

## S3 → ML preprocessing → database

```q
.qsp.run
    .qsp.v2.read.fromAmazonS3["s3://bucket/data.csv"; "eu-west-1"]
    .qsp.decode.csv[schema]
    .qsp.transform.replaceInfinity[cols]
    .qsp.transform.replaceNull[cols]
    .qsp.ml.minMaxScaler[cols]
    .qsp.v2.write.toDatabase[`mytable; .qsp.use `directWrite`database!(1b; "mydb")]
```

---

## Callback → console (debug / dev)

```q
.qsp.run .qsp.read.fromCallback[`pub] .qsp.write.toConsole[]
```

---

## Multiple sources (branching)

```q
trade: .qsp.read.fromCallback[`updTrade] .qsp.map[procTrade]
quote: .qsp.read.fromCallback[`updQuote] .qsp.map[procQuote]
.qsp.run (trade; quote)
```

---

## One source, multiple sinks (split)

```q
pipe: .qsp.read.fromCallback[`upd] .qsp.split[]
der: pipe .qsp.apply[derive]  .qsp.write.toStream[`derived]
raw: pipe .qsp.map[process]   .qsp.write.toStream[`raw]
.qsp.run (der; raw)
```
