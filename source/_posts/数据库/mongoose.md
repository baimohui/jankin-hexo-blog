---
title: egg.js 使用 mongoose
categories: 
- 数据库
tags:
- 数据库
- egg.js
- mongodb
- mongoose
---



### 1. 创建模型

```js
// app/model/visaOnArrivalModel.js
module.exports=app=>{
    const {mongoose}=app;
    const {Schema}=mongoose;
    const VisaOnArrivalSchema=new Schema(
        {
            OrderNumber:{type:String},
            FullName:{type:String},
            PassportNo:{type:String},
            DepartureFlightNumber:{type:String},
            TimeOfEntry:{type:String},
            ArriveAtTheAirport:{type:String},
            FlightNumber:{type:String},
            EnglishName:{type:String},
            Gender:{type:String},
            DateOfBirth:{type:String},
            Nationality:{type:String},
            PassportIssueDate:{type:String},
            PassportPeriodOfValidity:{type:String},
            DepartureDate:{type:String},
            DepartureCity:{type:String},
            Type:{type:String},
            Status:{type:String},
        },
        // 若想要在表中自动加入创建时间和修改时间可使用以下代码，timestamps里的值名称可以随意修改
        {
            timestamps: { createdAt: 'createAt', updatedAt: 'updateAt' }
        }
    );
    return mongoose.model("VisaOnArrivalModel",VisaOnArrivalSchema,"visaonarrivals")
}
/*注以上代码
定义了一张名为visaonarrivals的数据表，其中定义了多个键值名及其类型。（本例键值的数据类型为string类型）
mongoose中合法数据类型有:
*   String
*   Number
*   Date
*   Buffer
*   Boolean
*   Mixed
*   ObjectId
*   Array
*   Decimal128
*/	

```

<!--more-->

### 2. 查询数据

```js
// app/service/findVisaOnArrivalService.js
const Service = require("egg").Service;
class FVisaOnArrivalService extends Service {
    async FVisaOnArrival(obj) {
        const { ctx }=this;
        console.log(obj)
        //const res =await ctx.model.VisaOnArrivalModel.find({ArriveAtTheAirport:obj.ArriveAtTheAirport});
        const res = await ctx.model.VisaOnArrivalModel.find(obj);
        return res
    }
}
module.exports = FVisaOnArrivalService;

```

#### [`Model.find()`](https://mongoosejs.com/docs/api.html#model_Model.find)

```js
// With a JSON doc
Person.
  find({
    occupation: /host/,
    'name.last': 'Ghost',
    age: { $gt: 17, $lt: 66 },
    likes: { $in: ['vaporizing', 'talking'] }
  }).
  skip(10).
  limit(10).
  sort({ occupation: -1 }).
  select({ name: 1, occupation: 1 }).
  exec(callback);

// Using query builder
Person.
  find({ occupation: /host/ }).
  where('name.last').equals('Ghost').
  where('age').gt(17).lt(66).
  where('likes').in(['vaporizing', 'talking']).
  skip(10).
  limit(10).
  sort('-occupation').
  select('name occupation').
  exec(callback);

```



- [`Model.findById()`](https://mongoosejs.com/docs/api.html#model_Model.findById)
- [`Model.findByIdAndUpdate()`](https://mongoosejs.com/docs/api.html#model_Model.findByIdAndUpdate)
- [`Model.findOne()`](https://mongoosejs.com/docs/api.html#model_Model.findOne)
- [`Model.findOneAndReplace()`](https://mongoosejs.com/docs/api.html#model_Model.findOneAndReplace)
- [`Model.findOneAndUpdate()`](https://mongoosejs.com/docs/api.html#model_Model.findOneAndUpdate)
- [`Model.replaceOne()`](https://mongoosejs.com/docs/api.html#model_Model.replaceOne)
- [`Model.updateMany()`](https://mongoosejs.com/docs/api.html#model_Model.updateMany)
- [`Model.updateOne()`](https://mongoosejs.com/docs/api.html#model_Model.updateOne)





### 3. 修改数据

```js
// app/service/reviseService.js
const Service = require('egg').Service;
class reviseService extends Service {
    async ReviseDT(obj) {
        const { ctx } = this;
        let res = await ctx.model.VisaOnArrivalModel.findOneAndUpdate(
            { OrderNumber: obj.OrderID },
            { Status: obj.Status },
            function(err, data){
                if (err) {
                    return "修改失败"
                } else {
                    return "修改成功"
                }
            })
        return res
    }
}
module.exports = reviseService;
```



### 4. 删除数据

- [`Model.deleteMany()`](https://mongoosejs.com/docs/api.html#model_Model.deleteMany)
- [`Model.deleteOne()`](https://mongoosejs.com/docs/api.html#model_Model.deleteOne)
- [`Model.findByIdAndDelete()`](https://mongoosejs.com/docs/api.html#model_Model.findByIdAndDelete)
- [`Model.findByIdAndRemove()`](https://mongoosejs.com/docs/api.html#model_Model.findByIdAndRemove)
- [`Model.findOneAndDelete()`](https://mongoosejs.com/docs/api.html#model_Model.findOneAndDelete)
- [`Model.findOneAndRemove()`](https://mongoosejs.com/docs/api.html#model_Model.findOneAndRemove)

```js
// app/service/deleteService.js
const Service = require('egg').Service;
class DeleteService extends Service {
    async delete(obj){
        const { ctx } = this;
        let res = await ctx.model.VisaOnArrivalModel.deleteOne(
            { OrderNumber: obj.OrderID },
            function(err,data) {
                if(err) {
                    return "删除失败"
                } else {
                    return "删除成功"
                }
            })   
        return res
    }
}
module.exports = DeleteService
```



### 5. 添加数据

```js
// app/service/saveService.js
const Service = require('egg').Service;
class SaveService extends Service {
    async save(obj){
        const { ctx } = this;
        const user = new ctx.model.VisaOnArrivalModel({
            username: '圆号手',
            password: '我的密码',
        });
        await user.save()
    }
}
module.exports = DeleteService
```

