最近看到一个关于隐藏JavaScript代码的实现方案，挺有意思，主要通过零宽度字符的不可见特性来实现的。

# 代码实现

``` javascript
<script>
(function(window) {
  var rep = { // 替换用的数据，使用了4个零宽字符代理二进制
      '00': '\u200b',
      '01': '\u200c',
      '10': '\u200d',
      '11': '\uFEFF'
  };

  function hide(str) {
      //除Latin-1以外的字符替换成\uXXXX格式
      str = str.replace(/[^\x00-\xff]/g, function(a) {
          //escape(a)返回%uXXXXunicode编码
          return escape(a).replace('%', '\\');
      });

      str = str.replace(/[\s\S]/g, function(a) {
          // 转换成二进制
          a = a.charCodeAt().toString(2);
          //不足8位补0
          a = a.length < 8 ? Array(9 - a.length).join('0') + a : a;
          //../g表示每两位替换
          return a.replace(/../g, function(a) {
              return rep[a];
          });
      });
      return str;
  }

  //该字符串就是将@code按照hide的映射关系反查找到对应的二进制并替换
  var tpl = '("@code".replace(/.{4}/g,function(a){var rep={"\u200b":"00","\u200c":"01","\u200d":"10","\uFEFF":"11"};return String.fromCharCode(parseInt(a.replace(/./g, function(a) {return rep[a]}),2))}))';

  window.hider = function(code, type) {
      var str = hide(code); // 生成零宽字符串

      //此时str已经显示为一个“空”字符串了
      
      str = tpl.replace('@code', str); // 生成模版
      if (type === 'eval') {
          str = 'eval' + str;
      } else {
          str = 'Function' + str + '()';
      }

      return str;
  }
})(window);

</script>
```