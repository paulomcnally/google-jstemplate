Template processing is the staple pattern for separation of data and presentation in web applications. But it usually works on one page at a time, which is inadequate for incremental, asynchronous page updates typical of Ajax applications.

This system provides templates that allow for
  * incremental processing — every processing operation produces valid output, and
  * differential processing — output text is again a template.

It also fixes other undesirable properties of standard template processing:
  * Wellformed output is guaranteed.
  * Escaped by default.
  * Templates are intelligible: input template is valid output.

And, of course:
  * Pure javascript, HTML, browser side processing.