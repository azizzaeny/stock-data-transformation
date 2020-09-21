A series of learning functional programming by transforming product stock data. 


## Stock Data   
Task: Transform Data Into The desired Result    

#### Data 
```js

const productData = [
  {
    productId: 1000,
    productName: 'Product 1000'
  },
  {
    productId: 1001,
    productName: 'Product 1001'
  }
];

const locationData = [
  {
    locationId: 1,
    locationName: 'Location 1'
  },
  {
    locationId: 2,
    locationName: 'Location 2'
  }
];


const stockData = [
  {
    productId: 1000,
    locationId: 1,
    stock: 21
  },
  {
    productId: 1000,
    locationId: 2,
    stock: 8
  },
  {
    productId: 1001,
    locationId: 1,
    stock: 4
  },
  {
    productId: 1001,
    locationId: 2,
    stock: 10
  }
];
```

#### Result   
(expected)   

```js
const result = [
  {
    productName: 'Product 1000',
    stock: {
      total: 29,
      detail: [
        {
          locationName: 'Location 1',
          stock: 21
        },
        {
          locationName: 'Location 2',
          stock: 8
        }
      ]
    }
  },
  {
    productName: 'Product 1001',
    stock: {
      total: 14,
      detail: [
        {
          locationName: 'Location 1',
          stock: 4
        },
        {
          locationName: 'Location 2',
          stock: 10
        }
      ]
    }
  }
];
```


### Solution 1 - Minimum Builtin Function, No Sets, Maps, No Es6 spread.   
(Inefficient)      

Achieve the result with a minimum helper. the plan was to step from one transformation into another     

`(map) -> LocationName -> (groupBy) -> productId -> (concat) -> productDetail -> (sum) -> calculate total`   
   
   
#### Basic Function       
Recreating basic function map filter reduce to use.   
   
```js
const map = (fn, coll) =>{
  let buf = [];
  for(let x of coll) {
     buf.push(fn(x));
  }
  return buf;
}

const filter = (fn, coll) =>{
  let buf = [];
  for(let x of coll) {
      if(fn(x)){
          buf.push(x);
      }     
  };
  return buf;
}

const reduce = (fn, init, coll) =>{
  let acc = (init === undefined) ? 0 : init;
  for(let x of coll){
      if( acc !== undefined){
          acc = fn(acc, x);
      }else{
          acc = x;
      }
  }
  return acc;
}

const assign = (a,b) => (Object.assign(a,b));
const entries = (ob) => Object.entries(ob);
const first = (coll) => coll[0];
const second = (coll) => coll[1];
const concat = (a,b) => a.concat([b]);

const getKey = (key, fn, coll) =>{
  let index = coll.findIndex(x => fn(x));
  return coll[index][key];
}

const groupBy = (key, coll) =>{
  return reduce((acc, obj) =>{
      let value = obj[key];
      return (acc[value] = (acc[value] || []).concat(obj), acc);
  },{}, coll);
}

```

```js
//... basic functions above
const solution1 = (stockData, productData, locationData) =>{
  let stockDetail = (data) => x => ({
      productId : x.productId,
      stock     : x.stock,
      locationName : getKey('locationName', y => x.locationId == y.locationId, data)
     });
  
  let stockLocation = x =>({
      locationName: x.locationName,
      stock: x.stock 
  });
  
  let productSpec = (productName, detail) => ({
      productName: productName,
      stock:{
         detail: detail
      }});
    
  // step1
  let assignStockDetail = map(stockDetail(locationData), stockData);

  // step2
  let groupByProductId  = entries(groupBy('productId', assignStockDetail));

  // step3
  let concatProductDetail = reduce((acc, x) =>{
      let productId      = first(x),
          detailLocation = map(stockLocation, second(x)),
          productName    = getKey('productName', x => x.productId == productId, productData);

      return acc.concat([productSpec(productName, detailLocation)]);

  },[], groupByProductId);

  // step4
  let calculateTotal   = reduce((acc, x)=>{
      let res = assign(x,{});
      res.stock.total = reduce((acc, y)=> acc + y.stock, 0, x.stock.detail);
      return acc.concat([x]);
   }, [], concatProductDetail);
     
  return calculateTotal;
 };
 
```   




### Solution 2 - Single Reduction   
(Iterate over data once)    

The above method is inefficient because we loop and iterate over the array more and more, we need to find better solutions to this.    
as we can see from the above method that when we counting a stock total and aggregating the stock detail we can do it only once. making it presumable faster instead looping through all array.   

  
```js

const solution2 = (stockData, productData, locationData) =>{
  return Object.values(stockData.reduce((acc, value, index, coll) =>{
      let locationId    = value.locationId,
          stock         = value.stock,
          productId     = value.productId,
          locationName  = getKey('locationName', x => x.locationId == locationId, locationData),
          productName   = getKey('productName', x => x.productId == productId, productData),
          stockDetail   = {locationName, stock};

      if(!acc[productId]){
          acc[productId]={};
          acc[productId]['stock'] = {};
          acc[productId]['stock']['detail'] = [];
          acc[productId]['stock']['total']  = stock;
      }else{
          acc[productId]['stock']['total'] = acc[productId]['stock']['total'] + stock;
      }
          acc[productId]['productName'] = productName;
          acc[productId]['stock']['detail'] = acc[productId]['stock']['detail'].concat([stockDetail]);
      return acc;
  }, {}));
};                                           

```

#### Improved Solution 2  - Creating Indexed Hash Table Algorithm 
We can improve our algorithm by using hashTable, to make faster lookup data in our reference data `productData` and `locationData`

#### function indexHash
```js

class HashData {
  constructor(val){
    this.value = val;
  }
  deref(){
    return this.value;
  }
  indexBy(key){
    let curr = this.value;
	this.key = key;
	this.indexed = reduce((acc, obj) =>{
	  let value = obj[key];
	  return (acc[value] = (acc[value] || []).concat(obj), acc);
	},{}, curr);
  }
  get(k){
    return this.indexed[k][0];
  }
}

const createIndexHashTable = (key, data) =>{
  let res= new HashData(data);
  res.indexBy(key);
  return res;
}

const isIndexed = (x) => (x instanceof HashData);

const ensureIndexed = (x, key) => (x instanceof HashData) ? x : createIndexHashTable(key, x);

const fastLookup = (id, indexHash) => {
  if(!isIndexed(indexHash)){
      return 'data was not indexed';
  }
  return indexHash.get(id);
};

const hashValue = (indexHash) => indexHash.deref();

```

```js
//... functions indexHash

const solutions2b = (stockData,productData,locationData) =>{

  /* before cache the product and location into index table*/
  productData   = createIndexHashTable('productId', productData),
  locationData  = createIndexHashTable('locationId', locationData);

  /*...same as above*/

  /*example: without looping an array, get id from index*/ 
  let productName  = fastLookup(1000, productData),
      locationName = fastLookup(1, locationData);

  /*... same as above*/
  return {locationName, productName, productData, locationData};
  
}
```

### Solution 3 - Transducer Stream  

Although Solution 2 is working perfectly but at the cost of the code structure becoming ugly for developer point of view to read.    
Also, the mutation of js object doesn't look good because it modifies the previous object which would lead to complicated bugs and hard to trace.   

The next solution is to **"think of it as transformation over data and as a stream pipeline in a single iteration"**.    
 
So the end solutions would be use the transducer function. it efficient because we iterate reduction only once and not costing us for sluggish code to read.   



#### Creating transducer function  
By slightly modifying our Map, Filter, Reduce function to match with reducer code.  

Nb: we also can change the `getKeys` function and add our builded fast index lookup code in solutions 3,   

```js

const identity = (acc) => acc;

const push = () => [()=> [], (acc) => acc, (acc,x) => (acc.push(x), acc)];

const compose = (...fns) =>
fns.reduce((acc, val) => (...args) => val(acc(...args)), x => x);

const reduce = (reducer, initial, xs) => {
  let acc = initial != null ? initial : reducer[0]();
  for(let x of xs){
      acc = reducer[2](acc, x); 
  }
  return reducer[1](acc);
}


const transduce = (xform, rfn, initial, xs) => reduce(xform(rfn), initial, xs);


const map = (fn) => ([initial, completion, reducer]) => [
  initial, completion,
  (acc, x) => reducer(acc, fn(x))
];


const filter = (pred) => ([initial,completion, reducer]) =>{
  return [initial, completion, (acc, x) =>{
      return (pred(x) ? reducer(acc,x) : acc);
  }];
}


const partitionBy = (fn) => ([init, complete, reducer])=>{
  let sym =Symbol();
  let prev = sym, chunk = [];
  return [init, (acc) =>{
      if(chunk && chunk.length){
          acc=reducer(acc, chunk);
          chunk = [];
      }
      return complete(acc);
  }, (acc, x) =>{
      const curr = fn(x);
      if(prev === sym){
          prev = curr;
          chunk = [x];
      }else if(curr === prev){
          chunk.push(x);
      }else{
          chunk && (acc = reducer(acc, chunk));
          chunk = [x];
          prev = curr;
      }
      return acc;
  }];
}


const keySelector = (keys) =>{
  return (obj) => keys.reduce((acc, x) => {
      acc[x] = obj[x];
      return acc;
  }, {});
};

const selectKeys = (keys) => map(keySelector(keys));

var getKey = (key, fn, coll) =>{
  let index = coll.findIndex(x => fn(x));
  return coll[index][key];
}

const countKey = (keys, from, into) =>([init, completion, reducer]) =>{
  let seen = {};
  return  [
    init,
    completion,
    (acc,x) =>{
      let curr = x[keys];
      if(!seen[curr]){
        seen[curr] = 0;
      }
      seen[curr] = seen[curr] + x[from];
      x[into] = seen[curr];
      acc = reducer(acc, x);
      return acc;
    }
  ];
};

```

```js

const solution3 = (stockData, productData, locationData)=>{
    
   let addLocationName = (x) => ({ ...x, locationName: getKey('locationName', y => y.locationId == x.locationId, locationData) });

   let stockDataSummary = (x) => {
       let stockDetail = x.map(({locationName, stock}) => ({locationName, stock})),
           total       = Math.max.apply(Math,x.map(y => y._total)),
           sample      = x[0],
           productName = getKey('productName', y => y.productId == sample._id, productData);
           
       return {
           productName,
           stock:{
              total: total,
              detail: x
           }
       };
   };

   let stockId = (x) =>({
       _id : x.productId,
       ...x
    });
   
   let results = transduce(
       compose(
          map(stockDataSummary),
          partitionBy(x => x._id),
          countKey('_id', 'stock', '_total'),
          selectKeys(['locationName', 'stock', '_id'],),
          map(addLocationName),
          map(stockId),
          
       ), push(), [], stockData);
   return results;
  };

```


### Conclusions - All roads lead to rome   
The lesson to learn that there are many ways of producing the same results. it depends on how do you want to approach it. Thanks.   
