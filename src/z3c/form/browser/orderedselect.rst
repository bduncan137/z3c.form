
Ordered-Select Widget
---------------------

The ordered select widget allows you to select one or more values from a set
of given options and sort those options. Unfortunately, HTML does not provide
such a widget as part of its specification, so that the system has to use a
combnation of "option" elements, buttons and Javascript.

As for all widgets, the select widget must provide the new ``IWidget``
interface:

  >>> from zope.interface import verify
  >>> from z3c.form import interfaces
  >>> from z3c.form.browser import orderedselect

  >>> verify.verifyClass(interfaces.IWidget, orderedselect.OrderedSelectWidget)
  True

The widget can be instantiated only using the request:

  >>> from z3c.form import testing
  >>> request = testing.TestRequest()

  >>> widget = orderedselect.OrderedSelectWidget(request)

Before rendering the widget, one has to set the name and id of the widget:

  >>> widget.id = 'widget-id'
  >>> widget.name = 'widget.name'

We also need to register the template for at least the widget and request:

  >>> import zope.component
  >>> from zope.pagetemplate.interfaces import IPageTemplate
  >>> from z3c.form.testing import getPath
  >>> from z3c.form.widget import WidgetTemplateFactory

  >>> zope.component.provideAdapter(
  ...     WidgetTemplateFactory(getPath('orderedselect_input.pt'), 'text/html'),
  ...     (None, None, None, None, interfaces.IOrderedSelectWidget),
  ...     IPageTemplate, name=interfaces.INPUT_MODE)

If we render the widget we get an empty widget:

  >>> print(widget.render())
  <script type="text/javascript" src="++resource++orderedselect_input.js" />
  <table border="0" class="ordered-selection-field" id="widget-id">
    <tr>
      <td>
        <select id="widget-id-from" name="widget.name.from"
                size="5" multiple="multiple">
        </select>
      </td>
      <td>
        <button name="from2toButton" type="button"
                value="&rarr;"
                onclick="javascript:from2to('widget-id')">&rarr;</button>
        <br />
        <button name="to2fromButton" type="button"
                value="&larr;"
                onclick="javascript:to2from('widget-id')">&larr;</button>
      </td>
      <td>
        <select id="widget-id-to" name="widget.name.to"
                size="5" multiple="multiple">
        </select>
        <input name="widget.name-empty-marker" type="hidden" />
        <span id="widget-id-toDataContainer" style="display: none">
          <script type="text/javascript">
            copyDataForSubmit('widget-id');</script>
        </span>
      </td>
      <td>
        <button name="upButton" type="button" value="&uarr;"
                onclick="javascript:moveUp('widget-id')">&uarr;</button>
        <br />
        <button name="downButton" type="button" value="&darr;"
                onclick="javascript:moveDown('widget-id')">&darr;</button>
      </td>
    </tr>
  </table>

The json data representing the oredered select widget:
  >>> from pprint import pprint
  >>> pprint(widget.json_data())
  {'error': '',
   'id': 'widget-id',
   'label': '',
   'mode': 'input',
   'name': 'widget.name',
   'notSelected': (),
   'options': (),
   'required': False,
   'selected': (),
   'type': 'multiSelect',
   'value': ()}

Let's provide some values for this widget. We can do this by defining a source
providing ``ITerms``. This source uses descriminators wich will fit our setup.

  >>> import zope.schema.interfaces
  >>> from zope.schema.vocabulary import SimpleVocabulary
  >>> import z3c.form.term

  >>> class SelectionTerms(z3c.form.term.Terms):
  ...     def __init__(self, context, request, form, field, widget):
  ...         self.terms = SimpleVocabulary([
  ...              SimpleVocabulary.createTerm(1, 'a', u'A'),
  ...              SimpleVocabulary.createTerm(2, 'b', u'B'),
  ...              SimpleVocabulary.createTerm(3, 'c', u'C'),
  ...              SimpleVocabulary.createTerm(4, 'd', u'A'),
  ...              ])

  >>> zope.component.provideAdapter(SelectionTerms,
  ...     (None, interfaces.IFormLayer, None, None,
  ...      interfaces.IOrderedSelectWidget) )

Now let's try if we get widget values:

  >>> widget.update()
  >>> print(testing.render(widget, './/table//td[1]'))
  <td>
    <select id="widget-id-from" name="widget.name.from"
            size="5" multiple="multiple">
      <option value="a">A</option>
      <option value="b">B</option>
      <option value="c">C</option>
      <option value="d">A</option>
    </select>
  </td>

If we select item "b", then it should be selected:

  >>> widget.value = ['b']
  >>> widget.update()
  >>> print(testing.render(widget, './/table//select[@id="widget-id-from"]/..'))
  <td>
    <select id="widget-id-from" name="widget.name.from"
            size="5" multiple="multiple">
      <option value="a">A</option>
      <option value="c">C</option>
      <option value="d">A</option>
    </select>
  </td>

  >>> print(testing.render(widget, './/table//select[@id="widget-id-to"]'))
  <select id="widget-id-to" name="widget.name.to"
          size="5" multiple="multiple">
    <option value="b">B</option>
  </select>

The json data representing the oredered select widget:
  >>> from pprint import pprint
  >>> pprint(widget.json_data())
  {'error': '',
   'id': 'widget-id',
   'label': '',
   'mode': 'input',
   'name': 'widget.name',
   'notSelected': [{'content': 'A', 'id': 'widget-id-0', 'value': 'a'},
                   {'content': 'C', 'id': 'widget-id-2', 'value': 'c'},
                   {'content': 'A', 'id': 'widget-id-3', 'value': 'd'}],
   'options': [{'content': 'A', 'id': 'widget-id-0', 'value': 'a'},
               {'content': 'B', 'id': 'widget-id-1', 'value': 'b'},
               {'content': 'C', 'id': 'widget-id-2', 'value': 'c'},
               {'content': 'A', 'id': 'widget-id-3', 'value': 'd'}],
   'required': False,
   'selected': [{'content': 'B', 'id': 'widget-id-0', 'value': 'b'}],
   'type': 'multiSelect',
   'value': ['b']}

Let's now make sure that we can extract user entered data from a widget:

  >>> widget.request = testing.TestRequest(form={'widget.name': ['c']})
  >>> widget.update()
  >>> widget.extract()
  ('c',)

Unfortunately, when nothing is selected, we do not get an empty list sent into
the request, but simply no entry at all. For this we have the empty marker, so
that:

  >>> widget.request = testing.TestRequest(form={'widget.name-empty-marker': '1'})
  >>> widget.update()
  >>> widget.extract()
  ()

If nothing is found in the request, the default is returned:

  >>> widget.request = testing.TestRequest()
  >>> widget.update()
  >>> widget.extract()
  <NO_VALUE>

Let's now make sure that a bogus value causes extract to return the default as
described by the interface:

  >>> widget.request = testing.TestRequest(form={'widget.name': ['x']})
  >>> widget.update()
  >>> widget.extract()
  <NO_VALUE>

Finally, let's check correctness of widget rendering in one rare case when
we got selection terms with callable values and without titles. For example,
you can get those terms when you using the "Content Types" vocabulary from
zope.app.content.

  >>> class CallableValue(object):
  ...     def __init__(self, value):
  ...         self.value = value
  ...     def __call__(self):
  ...         pass
  ...     def __str__(self):
  ...        return 'Callable Value %s' % self.value

  >>> class SelectionTermsWithCallableValues(z3c.form.term.Terms):
  ...     def __init__(self, context, request, form, field, widget):
  ...         self.terms = SimpleVocabulary([
  ...              SimpleVocabulary.createTerm(CallableValue(1), 'a'),
  ...              SimpleVocabulary.createTerm(CallableValue(2), 'b'),
  ...              SimpleVocabulary.createTerm(CallableValue(3), 'c')
  ...              ])

  >>> widget.terms = SelectionTermsWithCallableValues(
  ...     None, testing.TestRequest(), None, None, widget)
  >>> widget.update()
  >>> print(testing.render(widget, './/table//select[@id="widget-id-from"]'))
  <select id="widget-id-from" name="widget.name.from"
          size="5" multiple="multiple">
    <option value="a">Callable Value 1</option>
    <option value="b">Callable Value 2</option>
    <option value="c">Callable Value 3</option>
  </select>
