
# DynamoDB Agent - Sources

*Sources utilisées pour la création de l'agent DynamoDB*

## Documentation Officielle

- **[AWS DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)** - Documentation officielle complète
- **[AWS DynamoDB API Reference](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/)** - Référence API officielle

## Articles et Ressources

### Design Patterns & Best Practices

- **[Three Tricks to Understanding Your DynamoDB Table](https://alexvipond.dev/blog/three-tricks-to-understanding-your-dynamodb-table)** - Alex Vipond
  - Trick 1: Comprendre la distribution des partitions
  - Trick 2: Analyser les patterns d'accès
  - Trick 3: Optimiser les clés de partition

### Key Design & Architecture

- **[Amazon DynamoDB Primary Key, Partition Key and Sort Key](https://medium.com/aws-lambda-serverless-developer-guide-with-hands/amazon-dynamodb-primary-key-partition-key-and-sort-key-how-to-choose-right-key-for-dynamodb-ea5673cb87c0)** - Mehmet Ozkaya
  - Différences entre clé primaire simple et composite
  - Fonction de hash pour déterminer la partition
  - Distribution uniforme des données

### Indexes & Performance

- **[DynamoDB Indexes Deep Dive](https://medium.com/@joudwawad/dynamodb-indexes-deep-dive-afe86a1cac48)** - Joud Wawad
  - GSI vs LSI : Différences et cas d'usage
  - Performance des index
  - Coût et optimisation
