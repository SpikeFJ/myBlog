# 导出
## 1. 导出json
`./mongoexport -h 127.0.0.1 -d A019 -c AREA_COLLECTION_FULL -o '../tmp20190708/1.json'`

## 2. 导出csv
`./mongoexport -h 127.0.0.1 -d A019 -c zb0_TMP --type csv -f _id,count -o '../20190718/zb0.csv'`

## 3. 导出指定数据
`./mongoexport -h 127.0.0.1 -d A019 -c SALEOBJECTIVES_VIEW_201907 --query '{"bw_plan":{$in:["0.0000"]}}' -o '../tmp20190711/TT.csv'`


# 导入
## 导入json
`./mongoimport -h 127.0.0.1 -d A019 -c AREA_COLLECTION_FULL --type json --file ../tmp20190708/1.json`

# 创建索引
`db.AREA_COLLECTION_FULL.createIndex({"personNo":1})`
`db.getCollection('AREA_COLLECTION_FULL').getIndexes()`

# 新增
`
db.artificial_field_value.insert({ 
    "field_text" : "销售计划",
"field_value" : "salePlan",
"field_type" : "task_type"
})
`
# 删除
`db.category_collection_day.remove({})`
`db.zb_dq1_TMP.drop()`

# 统计

`db.Tmp1.find({}).count()`

# 查询

## 1. 查询大区全量数据(指定用户名)
`db.getCollection('AREA_COLLECTION_FULL').find({ "personNo" : "18091161"})`

## 2. 查询大区增量数据(指定用户名)
`db.getCollection('AREA_COLLECTION_INSCREMENT').find({ "personNo" : "18091161"})`

## 3. 查询品类全量数据(指定用户名)
`db.getCollection('CATEGORY_COLLECTION_FULL').find({ "personNo" : "18091161"})`

## 4. 查询品类增量数据(指定用户名)
`db.getCollection('CATEGORY_COLLECTION_INSCREMENT').find({ "personNo" : "18091161"})`

## 5. 查询大区日数据
`db.getCollection('AREA_COLLECTION_DAY')find({ "personNo" : "18091161"})`

## 6. 查询品类日数据
`db.getCollection('CATEGORY_COLLECTION_DAY').find({ "personNo" : "18091161"})`

## 7. 查询元数据
`db.getCollection('artificial_field_value').find({ "personNo" : "18091161"})`


# 关联删除
`db.CATEGORY_COLLECTION_INSCREMENT_TMP.find({}).forEach(function(x){ db.CATEGORY_COLLECTION_INSCREMENT.remove({"_id":x._id}); })`

# 语句备份
`db.AREA_COLLECTION_FULL.find({}).forEach(function(x){ db.AREA_COLLECTION_FULL_20190716.insert(x); })`


# 单个字段分组
`
db.SALEOBJECTIVES_VIEW_201907.aggregate( [
{ $match : {"region_no":"","sub_co_code":"","store_code":""}},
{ $group: { _id:"$day",count:{$sum:1}} } 
] )
`     
# 多字段分组 
`                
db.SALEOBJECTIVES_VIEW_201907.aggregate( [
{ $match : {"region_no":"",