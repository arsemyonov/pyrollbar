**tl;dr:** Ideally it had to be done from scratch with
the backward compatibility api

# The existing pyrollbar

**Problem**

It's very dirty and only relies on global variables (settings, transformers, etc).

**Solution**

The best way to isolate the notifier is copy-paste code and put into class-level
implementation instead of module-level.


**Problem**

Selects environment (django, flask, twisted, etc) by brute forcing possible
imports without ability to use your own handler/adapter. The fun fact is that it's
hard to have multiple wsgi apps mount at once. Syntatic case: refactoring django→pyramid.

**Solution**

Add adapters to the class-level implementation that can be passed as an settings value.


# Proposal

Implement Notifier object that:

1. Could be instaniated anywhere and any time.
2. Could fork itself (#scope method) by passing new access_token and environmen.
3. Could Be extended by overriding (either inheritance or composition)
4. Could be extended by custom adapter implementation + DI/IoC.
5. Will not affect to global variables.

**Pros**

1. Easy to test. Mock, not patch.
2. Easy to make it thread-safe — for free.
3. No side-effects — no globals.

**Cons**

I don't see. Quite extensible.

The prototype is living in `redesign` branch.


# TODO

Make adapters that implement the following spec:
  - get_request_data (Common, Django, DRF, Flask, etc)
  - get_person_data
  - post
  — etc

Make a root notifier instance using the following spec:
  - report_message
  - report_exception
  - scope

Move all vendor related imports/discovery into `pyrollbar.contrib`.

As a good addition, condider `sys.excepthook`.

# Usage

```python
import rollbar

rollbar.configure('<access_token>', 'dev')
rollbar.report_message('root')

child = rollbar.scope('<access_token>', 'mailer')
child.report_message('child')

child.payload['extra'] = { 'key': 'value' }
child.report_message('child with extra')

rollbar.configure('<access_token', 'dev', adapter=TwistedAdapter)
rollbar.report_message('twisted message')

rollbar.configure('<access_token', 'dev', adapter=AgentAdapter)
rollbar.report_message('agent message')

child = rollbar.scope('<access_token>', 'environment')
child.adapter  # => <default adapter>


custom_adapter_child = rollbar.scope('<access_token>', 'environment', adapter=CustomAdapterImpl)
custom_adapter_child.adapter  # => <CustomAdapterImpl>
```
