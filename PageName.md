# Quick Example #
Here is an example template:
```
  <div id="t1">
    <h1 jscontent="title"></h1>
    <ul>
      <li jscontent="$this" jsselect="favs"></li>
    </ul>
  </div>
```
The template is just HTML with some special attributes. The $this variable refers to the data that template processing attaches to the `<li>` element (see the Processing Environment section of the Reference for details). Here is an example of javascript data this template can be used to display:
```
  var favdata = {
    title: 'Favorite Things', 
    favs: ['raindrops', 'whiskers', 'mittens']
  };
```
And here is a function that fills in the template with the data:
```
  function showData(data) {
    var input = new JsEvalContext(data);
    var output = document.getElementById('t1');
    jstProcess(input, output);
  }
```
Calling the above function (in, for example, an onload handler like `<body onload="showData(favdata)">`) produces the following structure:
```
  <div jstcache="0" id="t1">
    <h1 jstcache="1" jscontent="title">Favorite Things</h1>
    <ul jstcache="0">
      <li jsinstance="0" jstcache="2" jsselect="favs"
          jscontent="$this">raindrops</li>
      <li jsinstance="1" jstcache="3" jsselect="favs"
          jscontent="$this">whiskers</li>
      <li jsinstance="2" jstcache="4" jsselect="favs"
          jscontent="$this">mittens</li>
    </ul>
  </div>
```
Note that the special attributes are still there after the template has been processed. The filled template is still a template, and so it can be filled again (to show updates after the input data have changed, for example). The additional attributes jsinstance and jscache are used internally in order to make this reprocessing more efficient.
See the Reference for more details about all of the various "js" attributes in the template. [01-quick.html ](.md)