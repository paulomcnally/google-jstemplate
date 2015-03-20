# How to Use JsTemplate #

To use JsTemplate in an HTML document you will need to do four things:

include the JsTemplate javascript module and the modules on which it depends in your HTML document,
represent the data you want to display as a javascript object,
define the template or templates that will display your data, and
write the javascript to initiate template processing and display the output in your document.
## The JsTemplate Module ##

You will need a Subversion client to get JsTemplate. You may already have one (type `svn help` at the command line to check), but if you don't there are many Subversion client applications and plug-ins available.

Download the JsTemplate library (`jstemplate.js`) and its support files (`util.js` and `jsevalcontext.js`) using by executing the following Subversion command:

```
svn checkout http://google-jstemplate.googlecode.com/svn/trunk/ google-jstemplate-read-only
```
This command will create a `google-jstemplate-read-only` directory containing these three `.js` files, along with some examples. Any document that uses JsTemplate must link the `.js` files using `<script>` tags similar to the following:

```
<script src="util.js" type="text/javascript"></script> <script src="jsevalcontext.js" type="text/javascript"></script> <script src="jstemplate.js" type="text/javascript"></script>
```
## Javascript Data ##

A template can be used to display any javascript object. The template attributes determine how the object is mapped to an HTML representation. For a real application the data may initially come from the server, either as JSON data or in some other format that will need to be translated into a javascript object representation. For the sake of simplicity the examples in this document will initialize the data for template processing on the client-side by setting a variable. To make the data object available for template processing you will need to wrap it in an instance of the `JsEvalContext` class like so:
```
var input = new `JsEvalContext`(data);
```

> This class provides facilities for executing processing
instructions in an environment that includes `data` or some
portion of `data`, as explained in the Template Processing Instructions
Reference. The terms "data" and "data object" in this
documentation refer to the javascript object passed to a
`JsEvalContext` constructor, as in the above
statement.



## Template HTML ##

> HTML elements are marked for JsTemplate processing through
the addition of special attributes to tags. A "template" is simply an
element on which at least one of these special attributes is defined,
or an element containing descendants on which at least one of these
attributes is defined. This approach has the advantage that all
templates are valid HTML. For example, here is the template from the
Quick Example:



```
<div id="t1">
    <h1 jscontent="title"></h1>
    <ul>
    <li jscontent="$this" jsselect="favs"></li>
    </ul>
  </div>
```


We call element t1 a "template" because its children have the
JsTemplate processing instructions `jscontent` and
`jsselect` as attributes. These particular processing
instructions (described in greater detail below) tell
the Jst processor where to get the text content for a tag, and what
part of the data object will provide data for processing a subtree of
the template.




The output of template processing is also valid HTML, and preserves
the attributes of the input template. Here is the result of processing
the above template with the Quick Example data:



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


In other words, the output of JsTemplate processing is just another
template. This property allows the client-side updating unique to
JsTemplate. We can, for example, allow user actions to alter the
underlying data and immediately update the view. If our example
includes a link like:



```
  <a href="#" onclick="favdata.favs.push('packages');showData();">
    Reprocess
  </a>
```


then when we click on the "Reprocess" link the content of the template
becomes:


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
      <li jscontent="$this" jsselect="favs" jstcache="5" 
          jsinstance="3">packages</li>
    </ul>
  </div>
```

> [
> 01-quick.html
> ]


> Template HTML can appear in your document where you want to display
a representation of your data, but it doesn't have to. If you will be
reusing the same template structure in multiple places in a document,
you can include a single hidden copy of the template and then use
javascript to make new copies of its DOM node and place them in
multiple locations. This approach provides you with a kind of code
reuse for template HTML (see below).



## Processing Templates with Javascript Statements ##




The processing of a template is initiated with the javascript statement:



```
jstProcess(input, output);
```

> where `input` is a `JsEvalContext`
instance containing the data you want to display, and
`output` is a DOM Node instance for a template element. The
DOM Node for a template can be retrieved in the standard manner with
`document.getElementById(templateElementId)`.  The
JsTemplate library also provides a
`jstGetTemplate(templateElementId)` function which returns
a clone of the template element with id
`templateElementId`. This allows you to use a single
template as a pattern for multiple template instantiations. Even when
a template element will only be displayed once in a document, keeping
the original template hidden and unprocessed can be a useful design
methodology (see Advanced tips). To use
this approach you need to wrap your original template in a hidden
element, and then include an element that can be used as a peg on
which to hang a new copy of the template:



```
  <!--
  The element to which our template will be 
  attached at display-time:
  -->
  <div id="peg"></div>
  
  <!--
  A container to hide our template:
  -->
  <div style="display:none">
  
  <!--
  This is the template div. It will be copied 
  and attached to the div above.
  -->
  <div id="t1">
    <h1 jscontent="title"></h1>
    <ul><li jscontent="$this"
  jsselect="favs"></li></ul>
  </div>
  
  </div>
```


The first time you display your data, you can
use jstGetTemplate to get a copy of the template, clear the peg
element of all other content, hang the duplicate template on the peg,
and process the template:


```
var templateToProcess;
var peg = document.getElementById('peg');
       
templateToProcess = jstGetTemplate('t1');
// Clear the element to which we'll attach the template:
peg.innerHTML = '';
// Attach the template:
peg.appendChild(templateToProcess);
// and process it:
var processingContext = new `JsEvalContext`(favdata);
jstProcess(processingContext, templateToProcess);
```


Then when reprocessing the template you can just
use the peg element itself as your template by passing it to
jstProcess:


```
templateToProcess = peg;
var processingContext = new `JsEvalContext`(favdata);
jstProcess(processingContext, templateToProcess);
```