# Template Processing Instruction Reference #
## Overview ##
The processing instructions that define the results of jstProcess for a template are encoded as attributes in template HTML elements. There are eight such special attributes: `jsselect`, `jsdisplay`, ` jsskip`, ` jscontent`, `jsvars`, ` jsvalues, jseval`, and `transclude`. Before you dive into the details of individual instructions, however, you should know a little bit about the namespace within which these instructions are processed.

## Processing Environment ##
With the single exception of the `transclude` instruction, the values of all JsTemplate attributes will contain javascript expressions. These expressions will be evaluated in an environment that includes bindings from a variety of sources, and names defined by any of these sources can be referenced in Jst attribute expressions as if they were variables:


  * `JsEvalContext` data: All the properties of the `JsEvalContext`'s data object are included in the processing environment.
  * Explicitly declared variables: The setVariable(variableName, variableValue)` method of `JsEvalContext` creates a new variable with the name `variableName` in the processing environment if no such variable exists, assigning it the value `variableValue`. If the variable already exists, it will be reassigned the value `variableValue`. Variables can also be created and assigned with the jsvalues instruction (see below).

Note that variables defined in either of these ways are distinct from the `JsEvalContext` data object. Calling `setVariable` will not alter the data wrapped by the `JsEvalContext` instance. This fact can have important consequences when template processing is traversing the hierarchy of the data object (through the use of the `jsselect` instruction, for example -- see below): no matter what portion of the data hierarchy has been selected for processing, variables created with `setVariable` will always be available for use in template processing instructions.


  * Special variables: Jst also defines three special variables that can be used in processing instruction attributes:

  * `this`: The keyword `this` in JsTemplate attribute expressions will evaluate to the element on which the attribute is defined. In this respect JsTemplate attributes mirror event handler attributes like `onclick`.
  * `$index`: Array-valued data can result in a duplicate template node being created for each array element (see `jsselect`, below). In this case the processing environment for each of those nodes includes `$index` variable, which will contain the array index of the element associated with the node.
  * `$this`: `$this` refers to the `JsEvalContext` data object used in processing the current node. So in the above example we could substitute `$this.end` for `end` without changing the meaning of the `jscontent` expression. This may not seem like a very useful thing to do in this case, but there are other cases in which `$this` is necessary. If the `JsEvalContext` contains a value such as a string or a number rather than an object with named properties, there is no way to retrieve the value using object-property notation, and so we need `$this` to access the value.
So if you have the template

```
<div id="witha">
  <div id="Hey" 
       jscontent="this.parentNode.id + this.id + dataProperty + $this.dataProperty + declaredVar"
  ></div>
</div> 
```
and you process it with the statements

```
var mydata = {dataProperty: 'Nonny'};
var context = new `JsEvalContext`(mydata);
context.setVariable('declaredVar', 'Ho');
var template = document.getElementById('witha');
jstProcess(context, template); 
```
then the document will display the string `withaHeyNonnyNonnyHo`. [03-environ.html ](.md) The values of "id" and "parentNode.id" are available as properties of the current node (accessible through the keyword `this`), the value of `dataProperty` is available (via both a naked reference and the special variable `$this`) because it is defined in the `JsEvalContext`'s data object, and the value of `declaredVar` is available because it is defined in the `JsEvalContext`'s variables.

In the discussion of specific instruction attributes below, the phrase"current node" refers to the DOM element on which the attribute is defined.

## jscontent ##
This attribute is evaluated as a javascript expression in the current processing environment. The string value of the result then becomes the text content of the current node. So the template

```
<div id="tpl"> Welcome 
  <span jscontent="$this">
   (This placeholder name will be replaced by the actual username.) 
  </span>
</div> 
```
when processed with the javascript statements

```
var tplData = "Joe User";
var input = new `JsEvalContext`(tplData);
var output = document.getElementById('tpl');
jstProcess(input, output); 
```
will display

`Welcome Joe User`

in the browser. [04-jscontent.html ](.md) Note the use of `$this` here: the `JsEvalContext` constructor is passed the string "Joe User", and so this is the object to which `$this` refers.

When the Jst processor executes a `jscontent` instruction, a new text node object is created with the string value of the result as its nodeValue, and this new text node becomes the only child of the current node. This implementation ensures that no markup in the result is evaluated.

## jsselect ##
The primary function of JsTemplate is to create mappings between data structures and HTML representations of those data structures. The `jsselect` attribute handles much of the work of defining this mapping by allowing you to associate a particular subtree of the data with a particular subtree of the template's DOM structure. When a template node with a jsselect attribute is processed, the value of the jsselect attribute is evaluated as a javascript expression in the current `processing ` environment, as described above. If the result of this evaluation is not an array, the Jst processor automatically creates a new `JsEvalContext` object to wrap the result of the evaluation. The processing environment for the current node now uses this new `JsEvalContext` rather than the original `JsEvalContext`.

For example, imagine that you have the following data object (wrapped in a `JsEvalContext` object constructed with ```JsEvalContext``(tplData)`):

```
var tplData = { 
  username:"Jane User", 
  addresses:[ {
    location:"111 8th Av.",
    label:"NYC front door"
  }, {
    location:"76 9th Av.",
    label:"NYC back door"
  }, {
    location:"Mountain View",
    label:"Mothership"
  } ]
}; 
```
and you use this data in processing the template

```
<div id="tpl">
  <span jsselect="username" jscontent="$this"></span>'s Address Book </div> 
```
The `jsselect` attribute tells the processor to retrieve the username property of the data object, wrap this value ("Jane User") in a new `JsEvalContext`, and use the new `JsEvalContext` in processing the span element. As a result, `$this` refers to "Jane User" in the context of the span, and the `jscontent` attribute evaluates to "Jane User."

Note that the `jsselect` has to be executed before the `jscontent` in order for this example to work. In fact, `jsselect` is always evaluated before any other JsTemplate attributes (with the exception of `transclude`), and so the processing of all subsequent instructions for the same template element will take place in the new environment created by the `jsselect` (see "Order of Evaluation" below).

What happens if you try to jsselect the array-valued addresses property of the data object? If the result of evaluating a `jsselect` expression is an array, a duplicate of the current template node is created for each item in the array. For each of these duplicate nodes a new `JsEvalContext` will be created to wrap the array item, and the processing environment for the duplicate node now uses this new `JsEvalContext` rather than the original. In other words, `jsselect` operates as a sort of "for each" statement in the case of arrays. So you can expand your address book template to list the addresses in your data object like so:

```
<div id="tpl">
<h1><span jsselect="username" jscontent="$this">User de Fault</span>'s Address Book </h1>
<table cellpadding="5">
<tr>
  <td>
    <h2>Location:</h2>
  </td>
  <td>
    <h2> ;Label:</h2>
  </td>
</tr>
<tr jsselect="addresses">
  <td jscontent="location"></td>
  <td jscontent="label"></td>
</tr>
</table>
</div> 
```
Processing this template with your Jane User address book data will produce a nice table with a row for each address. [05-jsselect.html ](.md) Since the execution of a `jsselect` instruction can change the number of children under a template node, we might worry that if we try to reprocess a template with new data the template will no longer have the structure we want. JsTemplate manages this problem with a couple of tricks. First, whenever a `jsselect` produces duplicate nodes as a result of an array-valued expression, the Jst processor records an index for each node as an attribute of the element. So if the duplicate nodes are reprocessed, the processor can tell that they started out as a single node and will reprocess them as if they are still the single node of the original template. Second, a template node is never entirely removed, even if a jsselect evaluates to null. If a jsselect evaluates to null (or undefined), the current node will be hidden by setting "display='none'", and no further processing will be performed on it. But the node will still be present, and available for future reprocessing.

## jsdisplay ##
The value of the `jsdisplay` attribute is evaluated as a javascript expression. If the result is false, 0, "" or any other javascript value that is true when negated, the CSS display property of the current template node will be set to 'none', rendering it invisible, and no further processing will be done on this node or its children. This Jst instruction is particularly useful for checking for empty content. You might want to display an informative message if a user's address book is empty, for example, rather than just showing them an empty table. The following template will accomplish this goal:

```
<div id="tpl">
<h1><span jsselect="username" jscontent="$this">User de Fault</span>'s Address Book </h1>
<span jsdisplay="addresses.length==0">Address book is empty.</span> <table cellpadding="5" jsdisplay="addresses.length">
<tr>
  <td><h2>Location:</h2></td>
  <td><h2> ;Label:</h2></td>
</tr>
<tr jsselect="addresses">
  <td jscontent="location"></td>
  <td jscontent="label"></td>
</tr>
</table>
</div> 
```
If the addresses array is empty, the user will see "Address book is empty," but otherwise they will see the table of addresses as usual. [06-jsdisplay.html, 07-jsdisplay-empty.html ](.md)

## transclude ##
As `jsselect` does, the `transclude` instruction expands the structure of a template. Transclude does so by copying a structure from some other element in the document. The value of the `transclude` attribute is interpreted as an element id literal rather than as a javascript expression. (This difference, by the way, is the reason for the absence of a "js" in its name.)

If an element with the given id exists in the document, it is cloned and the clone replaces the node with the transclude attribute. Template processing continues on the new element. If no element with the given id exists, the node with the `transclude` attribute is removed. No further processing instruction attributes will be evaluated on a node if it has a `transclude` attribute.

The `transclude` attribute allows for recursion, because a template can be transcluded into itself. This feature can be handy when you want to display hierarchically structured data. If you have a hierarchically structured table of contents, for example, recursive `transclude` statements allow you represent the arbitrarily complex hierarchy with a simple template:

```
// Hierarchical data:
var tplData = {
  title: "JsTemplate",
  items: [ {
    title: "Using JsTemplate",
    items: [ { 
      title: "The JsTemplate Module"
    }, { 
      title: "Javascript Data"
    }, { 
      title: "Template HTML"
    }, { 
      title: "Processing Templates with Javascript Statements"
    } ]
  }, {
    title: "Template Processing Instructions",
    items: [ { 
      title: "Processing Environment" 
    }, { 
      title: "Instruction Attributes", 
      items: [ {
        title: "jscontent"
      }, {
        title: "jsselect"
      }, {
        title: "jsdisplay"
      }, {
        title: "transclude"
      }, {
        title: "jsvalues"
      }, {
        title: "jsskip"
      }, {
        title: "jseval"
      } ]
    } ]
  } ]
};
...
<div id="tpl">
<span jscontent="title">Outline heading</span>
<ul jsdisplay="items.length">
  <li jsselect="items">
    <!--Recursive tranclusion:-->
    <div transclude="tpl"></div>
  </li>
</ul>
</div> 
```
The recursion in this example terminates because eventually it reaches data objects that have no "items" property. When the `jsselect` asks for "items" on one of these leaves, it evaluates to null and no further processing will be performed on that node. Note also that when the node with a `transclude` attribute is replaced with the transcluded node in this example, the replacement node will not have a `transclude` attribute.

How to Use JsTemplate described the use of the `jstGetTemplate` function to process a copy of a template rather than the original template. Templates with recursive transcludes must be cloned in this way before processing. Because of the internal details of Jst processing, a template that contains a recursive reference to itself may be processed incorrectly if the original template is processed directly. The following javascript code will perform the required duplication for the above template:
```
var PEG_NAME = 'peg';
var TEMPLATE_NAME = 'tpl';
// Called by the body onload handler:
function jsinit() {
  pegElement = domGetElementById(document, PEG_NAME);
  loadData(pegElement, TEMPLATE_NAME, tplData);
}

function loadData(peg, templateId, data) {
  // Get a copy of the template:
  var templateToProcess = jstGetTemplate(templateId);
  // Wrap our data in a context object:
  var processingContext = new JsEvalContext(data);
  // Process the template
  jstProcess(processingContext, templateToProcess);
  // Clear the element to which we'll attach the processed template: 
  peg.innerHTML = '';
  // Attach the template:
  appendChild(peg, templateToProcess);
} 
```
[08-transclude.html ](.md)

## jsvalues ##
The `jsvalues` instruction provides a way of making assignments that alter the template processing environment. The template processor parses the value of the `jsvalues` attribute value as a semicolon-delimited list of name value pairs, with every name separated from its value by a colon. Every name represents a target for assignment. Every value will be evaluated as a javascript expression and assigned to its associated target. The nature of the target depends on the first character of the target name:


  * If the first character of the target name is "$", then the target name is interpreted as a reference to a variable in the current `JsEvalContext` processing environment. This variable is created if it doesn't already exist, and assigned the result of evaluating its associated expression. It will then be available for subsequent template processing on this node and its descendants (including subsequent name-value pairs in the same jsvalues attribute). Note that the dollar sign is actually part of the variable name: if you create a variable with `jsvalues="$varname:varvalue"`, you must use `$varname` to retrieve the value.


  * If the first character of the target name is ".", then the target name is interpreted as a reference to a javascript property of the current template node. The property is created if it doesn't already exist, and is assigned the result of evaluating its associated expression. So the instruction `jsvalues=".id:'Joe';.style.fontSize:'30pt'"` would change the id of the current template node to "Joe" and change its font size to 30pt. [09-jsvalues.html ](.md)


  * If the first character of the target name is neither a dot nor a dollar sign, then the target name is interpreted as a reference to an XML attribute of the current template element. In this case the instruction `jsvalues="name:value"` is equivalent to the javascript statement `this.setAttribute('name','value')`, where `this` refers to the current template node. Just as in the case of a call to `setAttribute`, the value will be interpreted as a string (after javascript evaluation). So `jsvalues="sum:1+2"` is equivalent to `this.setAttribute('sum', '3')`.

The `jsvalues` instruction makes a handy bridge between the DOM and Jst data. If you want a built-in event handler attribute like `onclick` to be able to access the currently selected portion of the `JsEvalContext` data, for example, you can use `jsvalues` to copy a reference to the data into an attribute of the current element, where it will be accessible in the `onclick` attribute via `this`. The following example uses this approach to turn our outline into a collapsible outline:

```
// Function called by onclick to record state of closedness and
// refresh the outline display
function setClosed(jstdata, closedVal) {
  jstdata.closed = closedVal;
  loadData(PEG_ELEMENT, TEMPLATE_NAME, tplData);
}
... 
<div id="tpl"> <!-- Links to open and close outline sections: -->
<a href="#" jsdisplay="closed" 
 jsvalues=".jstdata:$this"
 onclick="setClosed(this.jstdata,0)">[Open]</a>
<a href="#" jsdisplay="!closed && items.length"
 jsvalues=".jstdata:$this"
 onclick="setClosed(this.jstdata,1)">[Close]</a>
<span jscontent="title">Outline heading</span>
<ul jsdisplay="items.length && !closed">
  <li jsselect="items">
    <!--Recursive tranclusion:-->
    <div transclude="tpl"></div>
  </li>
</ul>
</div> 
```
[10-jsvalues.html ](.md)

## jsvars ##
This instruction is identical to jsvalues, except that all assignment targets are interpreted as variable names, whether or not they start with a "$." That is, all assignment targets are interpreted as described in section 1 of the `jsvalues` section above.

## jseval ##
The Jst processor evaluates a `jseval` instruction as a javascript expression, or a series of javascript expressions separated by semicolons. The `jseval` instruction thus allows you to invoke javascript functions during template processing, in the usual template processing environment, but without any of the predefined template processing effects of `jsselect`, `jsvalues`, `jsdisplay`, `jsskip`, or `jscontent`. For example, with the addition of a `jseval` instruction to our outline title span, the Jst processor can record a count of the total number of outline items with and without titles as it traverses the data hierarchy.

The count information in this example is stored in the processing context with a call to `setVariable`, so that it will be available to template processing throughout the data hierarchy:

```
processingContext.setVariable('$counter', counter); 
```
A `jseval` expression increments the count:

```
<span jscontent="title"
  jseval="title? $counter.full++: $counter.empty++"> Outline heading </span> 
```
and then a separate template displays these counts at the bottom of the page:

```
<div id="titleCountTpl">
<p>This outline has <span jscontent="$counter.empty"></span>
empty titles and <span jscontent="$counter.full"></span>
titles with content.</p>
</div> 
```
Note that when you close headings the counts change: `jsdisplay` is not only hiding the closed elements, but also aborting the processing of these elements, so that the `jseval` expressions on these elements are never evaluated. [11-jseval.html ](.md)

## jsskip ##
The value of the `jsskip` attribute is evaluated as a javascript expression. If the result is any javascript value that evaluates to `true` in a boolean context, then the Jst processor will not process the subtree under the current node. This instruction is useful for improving the efficiency of an application (to avoid unnecessarily processing deep trees, for example).

The effect of a `jsskip` that evaluates to `true` is very similar to the result of a `jsdisplay` that evaluates to false. In both cases, no processing will be performed on the node's children. `Jsskip` will not, however, prevent the current node from being displayed.

## Order of Evaluation ##
JsTemplate instruction attributes within a single element are evaluated in the following order:


  * `transclude`. If a transclude attribute is present no further Jst attributes are processed.
  * `jsselect`. If `jsselect` is array-valued, remaining attributes will be copied to each new duplicate element created by the `jsselect` and processed when the new elements are processed.
  * `jsdisplay`
  * `jsvars`
  * `jsvalues`
  * `jseval`
  * `jsskip`
  * `jscontent`