
### Example 1: Range + equality query , compound index is suitable:

```js
db.a.drop();
db.a.insertMany([{a:10,b:10,c:30},{a:15,b:20,c:30},{a:10,b:4,c:100},{a:1,b:4,c:100},{a:80,b:33,c:90},{a:180,b:40,c:110}]);
x=db.a.find();
print(x);

db.a.createIndex({a:1,b:1},{name:"ub"})

print(
    db.a.aggregate([{$match:{a:{$gt:20}}},{$match:{b:33}}], {hint:"ub",explain:true})
);
```
#### Result:
```
  executionStats: {
    executionSuccess: true,
    nReturned: 1,
    executionTimeMillis: 0,
    totalKeysExamined: 2,
    totalDocsExamined: 1,
```

### Example 2: Range + equality query , compound index covers only the range part:

```js
db.b.drop();
db.b.insertMany([{a:10,b:10,c:30},{a:15,b:20,c:30},{a:10,b:4,c:100},{a:1,b:4,c:100},{a:8,b:33,c:90},{a:18,b:40,c:110}]);
x=db.b.find();
print(x);

db.b.createIndex({a:1,b:1},{name:"ub"})

print(
    db.b.aggregate([{$match:{a:{$gt:1}}},{$match:{c:100}}], {hint:"ub",explain:true})
);

print(
    db.b.aggregate([{$match:{a:{$gt:1}}},{$match:{c:100}}], {hint:"ub"})
);
```

#### Result: index prevents reading one doc out of 6, but within that range all 5 docs need to be fetched and examined:
```
  executionStats: {
    executionSuccess: true,
    nReturned: 1,
    executionTimeMillis: 0,
    totalKeysExamined: 5,
    totalDocsExamined: 5,
```

```
[ { _id: ObjectId("654942bdb610d9bdbf4321de"), a: 10, b: 4, c: 100 } ]
```


### Example 3: A fully covered range query: index contains all the data, no reading from disk. Note: `, _id:0`

```js
db.c.drop();
db.c.insertMany([{a:10,b:10,c:30},{a:15,b:20,c:30},{a:21,b:4,c:100},{a:1,b:4,c:100},{a:8,b:33,c:90},{a:18,b:40,c:110}]);
x=db.c.find();
print(x);

db.c.createIndex({a:1},{name:"ub"})

print(
    db.c.aggregate([{$match:{a:{$gt:2}}},{$project:{a:1, _id:0}}], {hint:"ub",explain:true})
);
print(
    db.c.aggregate([{$match:{a:{$gt:2}}},{$project:{a:1, _id:0}}], {hint:"ub"})
);
```

#### Result: Entire result is coming from the index:
```
  executionStats: {
    executionSuccess: true,
    nReturned: 5,
    executionTimeMillis: 0,
    totalKeysExamined: 5,
    totalDocsExamined: 0,
```

```
[ { a: 8 }, { a: 10 }, { a: 15 }, { a: 18 }, { a: 21 } ]
```


### Example 4: ESR (Equality - Sort -Range) example which is the ideal compound index design:

```js
db.esr.drop();
db.esr.insertMany([{a:10,b:10,c:30},{a:10,b:50,c:95},{a:21,b:4,c:100},{a:10,b:4,c:100},{a:10,b:33,c:90},{a:10,b:40,c:110}]);
x=db.esr.find();
print(x);

// we design the index for a query that: 1. matches equality on a, 2. sorts on b and 3. applies a range on c:
db.esr.createIndex({a:1,b:1,c:1},{name:"ub"})

// a query that: 1. matches equality on a, 2. sorts on b and 3. applies a range on c:
print(
    db.esr.aggregate([{$match:{a:10}},{$sort:{b:1}},{$match:{c:{$gt:90}}}], {hint:"ub",explain:true})
);
print(
    db.esr.aggregate([{$match:{a:10}},{$sort:{b:1}},{$match:{c:{$gt:90}}}], {hint:"ub"})
);
```

#### Result: Only need to fetch the 3 docs that are being returned and the result is already sorted by b automatically (based on the index), no in memory sort is needed. The range query on c can be done by examining the index, nReturned=totalDocsExamined:
```
  executionStats: {
    executionSuccess: true,
    nReturned: 3,
    executionTimeMillis: 0,
    totalKeysExamined: 6,
    totalDocsExamined: 3,
```

```
[
  { _id: ObjectId("65494815cf6f0b937f1bc964"), a: 10, b: 4, c: 100 },
  { _id: ObjectId("65494815cf6f0b937f1bc966"), a: 10, b: 40, c: 110 },
  { _id: ObjectId("65494815cf6f0b937f1bc962"), a: 10, b: 50, c: 95 }
]
```



### Example 5: Taking previous ESR example one step further: Make the query fully covered by index by removing the _id from result:

```js
db.esr.drop();
db.esr.insertMany([{a:10,b:10,c:30},{a:10,b:50,c:95},{a:21,b:4,c:100},{a:10,b:4,c:100},{a:10,b:33,c:90},{a:10,b:40,c:110}]);
x=db.esr.find();
print(x);

// we design the index for a query that: 1. matches equality on a, 2. sorts on b and 3. applies a range on c:
db.esr.createIndex({a:1,b:1,c:1},{name:"ub"})

// a query that: 1. matches equality on a, 2. sorts on b and 3. applies a range on c:
print(
    db.esr.aggregate([{$match:{a:10}},{$sort:{b:1}},{$match:{c:{$gt:90}}},{$project:{a:1,b:1,c:1, _id:0}}], {hint:"ub",explain:true})
);
print(
    db.esr.aggregate([{$match:{a:10}},{$sort:{b:1}},{$match:{c:{$gt:90}}},{$project:{a:1,b:1,c:1, _id:0}}], {hint:"ub"})
);
```

#### Result: Only need to fetch the 3 docs that are being returned and the result is already sorted by b automatically (based on the index), no in memory sort is needed. The range query on c can be done by examining the index, nReturned=totalDocsExamined:
```
  executionStats: {
    executionSuccess: true,
    nReturned: 3,
    executionTimeMillis: 0,
    totalKeysExamined: 6,
    totalDocsExamined: 0,
```

```
[
  { a: 10, b: 4, c: 100 },
  { a: 10, b: 40, c: 110 },
  { a: 10, b: 50, c: 95 }
]
```
