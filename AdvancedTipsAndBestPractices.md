# Advanced Tips and Best Practices #

## Using Templates to Create a Design Glossary ##
As discussed in "Processing Templates with Javascript Statements" above, the function `jstGetTemplate` returns a clone of a template, making it possible to use the original template as a sort of factory for generating multiple elements with identical layouts. In some cases (recursive transcludes) this approach is necessary, but one could argue that the approach is always useful. If you keep all of your templates in a hidden (`display="none"`) element of your HTML document, you end up with a glossary of design elements built into your document. During the design phase, then, you can make your templates visible in order to see the entire vocabulary of available elements. Because your templates are valid HTML, they can include sample text or HTML to illustrate the results of executing `jscontent` instructions. When the templates are copied and processed, the sample text will be replaced with the `jscontent` results.

## Using Cached `JsEvalContext` Instances ##
Creating new instances can be an expensive process, particularly in Internet Explorer 6. This fact could have been a problem for the performance of JsTemplate, since every `jsselect` instruction requires the use of a new `JsEvalContext` object to represent the new subset of data being selected. To deal with this potential difficulty the Jst processor maintains a cache of `JsEvalContext` instances, reusing old instances rather than constructing a new instance for every `jsselect`. The functions used to acquire `JsEvalContext`s from the cache and to return them to the cache for reuse are also available to Jst users. You may want to consider constructing `JsEvalContext`s in this way, particularly if your code is frequently calling the `JsEvalContext` constructor. To obtain a `JsEvalContext` instance, you can call:


```
var context = JsEvalContext.create(data);
```
Then when you are done with the instance (when, for example, you need to reprocess the template with an entirely new set of data) you can call:

```
JsEvalContext.recycle(context);
```
This call will erase the data wrapped by the instance and add it to the pool of `JsEvalContext`s available for reuse.

## Using the Parent Argument to `JsEvalContext` ##
The examples of the `JsEvalContext` constructor presented above take a single argument: the data to be wrapped by the `JsEvalContext` instance. This constructor can, however, also take an optional second argument: a second `JsEvalContext` object to serve as a "parent" for the instance being constructed. If this second argument is provided, all variables will be copied from the parent to the newly constructed instance.

Imagine, for example, that you have two different sets of data (an address book and a list of recent searches, for example), each with its own template, but you also have some data that you need to present in both the templates (the user's name, for example). You only want to store the shared data in one place: if the data is updated you don't want to have to worry about keeping it current in two separate places. By using the second argument to `JsEvalContext` you can keep the shared data in its own `JsEvalContext` and pass that single instance to the constructors for the other two context objects:


```
<script type="text/javascript">
var user = "Jane User";
var tpl1Data = {addresses:[ {location:"111 8th Av.", label:"NYC front door"}, {location:"76 9th Av.", label:"NYC back door"}, {location:"Mountain View", label:"Mothership"} ] }; var tpl2Data = {addresses:[ {location:"534 Carlton Ave."}, {location:"772 Broadway"} ] };
function showData() { // This is the javascript code that processes the template:
var parent = new JsEvalContext();
parent.setVariable('username', user);
var input1 = new JsEvalContext( tpl1Data, parent);
var output1 = document.getElementById('tpl1');
jstProcess(input1, output1);
var input2 = new JsEvalContext( tpl2Data, parent);
var output2 = document.getElementById('tpl2');
jstProcess(input2, output2); } </script>
...
<div id="tpl1"> <h1>
<span jsselect="username" jscontent="$this">User de Fault</span>'s Address Book </h1> <table cellpadding="5">
<tr><td><h2>Location:</h2></td><td><h2>Label:</h2></td></tr>
<tr jsselect="addresses"><td jscontent="location"></td> <td jscontent="label"></td></tr> </table></div>
<div id="tpl2"> <h1 jsselect="username" jscontent="$this + '\'s Previous Searches'"></h1>
<ul>
  <li jsselect="addresses" jscontent="location"></li>
</ul>
</div> 
```