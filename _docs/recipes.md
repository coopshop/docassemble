---
layout: docs
title: Recipes
short_title: Recipes
---

This section contains miscellaneous recipes for solving problems in
**docassemble**.

# <a name="require checkbox"></a>Require a checkbox to be checked

## Using `validation code`

{% highlight yaml %}
modules:
  - docassemble.base.util
---
question: |
  You must agree to the terms of service.
fields:
  - I agree to the terms of service: agrees_to_tos
    datatype: yesnowide
validation code: |
  if not agrees_to_tos:
    validation_error("You cannot continue until you agree to the terms of service.")
---
mandatory: True
need: agrees_to_tos
question: All done.
{% endhighlight %}

## Using `datatype: checkboxes`

{% highlight yaml %}
question: |
  You must agree to the terms of service.
fields:
  - no label: agrees_to_tos
    datatype: checkboxes
    minlength: 1
    choices:
      - I agree to the terms of service
    validation messages:
      minlength: |
        You cannot continue unless you check this checkbox.
---
mandatory: True
need: agrees_to_tos
question: All done
{% endhighlight %}

# <a name="completion"></a>Use a variable to track when an interview has been completed

One way to track whether an interview is completed is to set a
variable when the interview is done.  That way, you can inspect the
interview answers and test for the presence of this variable.

{% highlight yaml %}
modules:
  - docassemble.base.util
---
objects:
  - user: Individual
---
question: |
  What is your name?
fields:
  - First name: user.name.first
  - Last name: user.name.last
---
mandatory: True
code: |
  user.name.first
  user_finish_time
  final_screen
---
code: |
  user_finish_time = current_datetime()
---
event: final_screen
question: |
  Goodbye, user!
buttons:
  Exit: exit
{% endhighlight %}

You could also use Redis to store the status of an interview.

{% highlight yaml %}
modules:
  - docassemble.base.util
---
objects:
  - user: Individual
  - r: DARedis
---
question: |
  What is your name?
fields:
  - First name: user.name.first
  - Last name: user.name.last
---
mandatory: True
code: |
  interview_marked_as_started
  user.name.first
  interview_marked_as_finished
  final_screen
---
code: |
  redis_key = user_info().filename + ':' + user_info().session
---
code: |
  r.set(redis_key, 'started')
  interview_marked_as_started = True
---
code: |
  r.set(redis_key, 'finished')
  interview_marked_as_finished = True
---
event: final_screen
question: |
  Goodbye, user!
buttons:
  Exit: exit
{% endhighlight %}

# <a name="exit hyperlink"></a>Exit interview with a hyperlink rather than a redirect

Suppose you have a final screen in your interview that looks like this:

{% highlight yaml %}
modules:
  - docassemble.base.util
---
mandatory: True
code: |
  kick_out
---
event: kick_out
question: Bye
buttons:
  - Exit: exit
    url: https://example.com
{% endhighlight %}

When the user clicks the "Exit" button, an Ajax request is sent to the
**docassemble** server, the interview logic is run again, and then
when the browser processes the response, the browser is redirected
by JavaScript to the url (https://example.com).

If you would rather that the button act as a hyperlink, where clicking
the button sends the user directly to the URL, you can make the button
this way:

{% highlight yaml %}
modules:
  - docassemble.base.util
---
mandatory: True
code: |
  kick_out
---
event: kick_out
question: Bye
subquestion: |
  ${ action_button_html("https://example.com", size='md', color='primary', label='Exit', new_window=False) }
{% endhighlight %}

# <a name="two fields match"></a>Ensure two fields match

{% highlight yaml %}
modules:
  - docassemble.base.util
---
question: |
  What is your e-mail address?
fields:
  - E-mail: email_address_first
    datatype: email
  - note: |
      Please enter your e-mail address again.
    datatype: email
  - E-mail: email_address
    datatype: email
  - note: |
      Make sure the e-mail addresses match.
    js hide if: |
      val('email_address') != '' && val('email_address_first') == val('email_address')
  - note: |
      <span class="text-success">E-mail addresses match!</span>
    js show if: |
      val('email_address') != '' && val('email_address_first') == val('email_address')
validation code: |
  if email_address_first != email_address:
    validation_error("You cannot continue until you confirm your e-mail address")
---
mandatory: True
question: |
  Your e-mail address is ${ email_address }.
{% endhighlight %}

# <a name="progressive disclosure"></a>Progresive disclosure

{% include demo-side-by-side.html demo="progressive-disclosure" %}

Add `progressivedisclosure.css` to the "static" data folder of your package.

{% highlight css %}
a span.pdcaretopen {
    display: inline;
}

a span.pdcaretclosed {
    display: none;
}

a.collapsed .pdcaretopen {
    display: none;
}

a.collapsed .pdcaretclosed {
    display: inline;
}
{% endhighlight %}

Add `progressivedisclosure.py` as a Python module file in your
package.

{% highlight python %}
import re

__all__ = ['prog_disclose']

def prog_disclose(template, classname=None):
    if classname is None:
        classname = ' bg-light'
    else:
        classname = ' ' + classname.strip()
    the_id = re.sub(r'[^A-Za-z0-9]', '', template.instanceName)
    return u"""\
<a class="collapsed" data-toggle="collapse" href="#{}" role="button" aria-expanded="false" aria-controls="collapseExample"><span class="pdcaretopen"><i class="fas fa-caret-down"></i></span><span class="pdcaretclosed"><i class="fas fa-caret-right"></i></span> {}</a>
<div class="collapse" id="{}"><div class="card card-body{} pb-1">{}</div></div>\
""".format(the_id, template.subject_as_html(trim=True), the_id, classname, template.content_as_html())
{% endhighlight %}

This uses the [collapse feature] of [Bootstrap].

# <a name="new or existing"></a>New object or existing object

The [`object` datatype] combined with the [`disable others`] can be
used to present a single question that asks the user either to select
an object from a list or to enter information about a new object.

Another way to do this is to use `show if` to show or hide fields.

This recipe gives an example of how to do this in an interview that
asks about individuals.

{% include demo-side-by-side.html demo="new-or-existing" %}

This recipe keeps a master list of individuals in an object called
`people`.  Since this list changes throughout the interview, it is
re-calculated whenever a question is asked that uses `people`.

When individuals are treated as unitary objects, you can do things
like use Python's `in` operator to test whether an individual is a
part of a list.  This recipe illustrates this by testing whether
`boss` is part of `customers` or `employee` is part of `customers`.

# <a name="resume via email"></a>E-mailing the user a link for resuming the interview later

If you want users to be able to resume their interviews later, but you
don't want to use the username and password system, you can e-mail
your users a URL created with [`interview_url()`].

{% include demo-side-by-side.html demo="save-continue-later" %}

[collapse feature]: https://getbootstrap.com/docs/4.0/components/collapse/
[Bootstrap]: https://getbootstrap.com/
[`disable others`]: {{ site.baseurl }}/docs/fields.html#disable others
[`object` datatype]: {{ site.baseurl }}/docs/fields.html#object
[`interview_url()`]: {{ site.baseurl }}/docs/functions.html#interview_url