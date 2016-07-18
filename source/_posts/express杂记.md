title: express杂记
date: 2016-07-16 23:02:39
tags: [nodejs]
---
app.set/get
---
用来查看或者设置一些配置参数



express里变量的换行出不来
---
```javascript
router.get('/get_log', function (req, res) {
      worker.exec("cat " + req.query.location, function (error, stdout, stderr) {
        console.log('stderr : ' + stderr);
        console.log('stdout : ' + stdout);
        console.log('error : ' + error);
        res.render('etls/log', {stdout : stdout, log : req.query.location});
      }); 
    });
```
后面控制台的输出
```
stderr : 
stdout : # job for mymeta@my_table
# author : will
# create time : 19700118070840
# pre settings 
delete from default.tbl
select * from batting
error : null

```
模板文件
```
#{stdout}
```

前端输出
```
# job for mymeta@my_table # author : will # create time : 19700118070840 # pre settings delete from default.tbl select * from batting
```
控制台的stdout变量是换行的，但是前端就不换行。试了一下res.send()，结果也是这样。

模板修改为
```jade
!{stdout.replace(/\n/g, '<br/>')}
```

如果使用res.send()的话，就res.send(stdout.replace(/\n/g, '<br/>'))就行了。

这个换行支持起来那么难么？比较low的感觉。



body-parser
---
安装了body parser之后就可以方便的在req中使用 `req.body`来获取form内容，使用`req.query`来获取querystring里的内容了。

但是一定要注意express的app.use是有前后次序的。`踩过坑`

