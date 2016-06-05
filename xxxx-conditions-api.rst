=================================
DEP ----: Conditions API
=================================

:DEP: --UNASSIGNED--
:Author: Connor Boyle
:Status: Draft
:Type: Feature
:Created: 2016-05-09
:Last-Modified: 2016-05-09
:Django-Version: 1.??

.. contents:: Table of Contents
   :depth: 3
   :local:

Team
====

Author
    - Connor Boyle (cboyle at macalester dot e.d.u.)

Implementation Team
    - Connor Boyle
    - *Seeking implementors*

Shepherd
    - *Seeking shepherd*


Motivation
==========

In its current form, Django gives developers no standardized mechanism for
providing their project with consistent definitions of dynamic conditions (i.e.
function-backed permissions). As a result of this, Django also does not
provide developers with standardized ways of defining dynamic conditions or
inserting them into the constructs that developers often extend (View, Form,
ModelAdmin, etc.). Even worse, the modifications needed to make these
constructs check conditions are often complicated and un-intuitive to the point
of forcing the developer to intimately acquaint themself with the internal
implementations of these pieces of helper code–thus defeating their purpose as
abstractions.

Abstract
========

This DEP would introduce a new concept to Django; the concept of the Condition.
A Condition, simply put, would be a definition–in plain Python–of a set of
rules and requirements relative to the values of certain keyword arguments,
defined explicitly by the developer. The primary purpose of these Conditions
will be to serve as "dynamic permissions"; i.e., permissions based on whether
certain requirements involving the user and other variables related to the
request are met, rather than a value in a database.

Like–for example–class-based views or the Forms API, this won't provide any
functionality a smart developer *couldn't* have achieved on their own. It would
however, provide a common standard and framework for doing so quickly, easily,
and consistently.

Potential Solutions from Other Packages
=======================================

A few third-party packages have attempted to solve this problem. Below are
outlined their methods, as well as their key differences with this proposal.

Django Rest Framework
---------------------

`Django Rest Framework <http://www.django-rest-framework.org/>`_, a package for
building RESTful web APIs (generally for AJAX), also provides its own `dynamic
authorization system
<http://www.django-rest-framework.org/api-guide/permissions/>`_.

One inconsistency in this framework is that the developer is encouraged to base
the results of these dynamic permissions on the method of the request (and
therefore to make assumptions about what a particular method "does"). This is
evidenced by some of the default "Permissions" provided by the package, namely
`IsAuthenticatedOrReadOnly
<http://www.django-rest-framework.org/api-guide/permissions/#isauthenticatedorreadonly>`_.
The problem with this design pattern (authentication based on HTTP method) is
that it includes information about an action to determine whether a user is
allowed to perform an application, breaking clean encapsulation that otherwise
separates action from rule throughout the framework.

Django Rest Framework's permissions' encapsulation is further limited in that
their rule-checking method must take ``view`` as an argument, thus limiting
each Permission's usability to authenticating in the context of a view (as
opposed to, say, in a template to determine whether a link should be
displayed).

Django Rules
------------

`Django-Rules <https://github.com/dfunckt/django-rules>`_ (indeed it does)
provides a framework for the sole purpose of defining and implementing these
dynamic permissions, which it calls "rules". The package's inner workings tie
it closely to Django's default database-backed permissions system; the package
essentially attempts to directly replace Django's database-backed permissions
with developer-defined method-backed permissions. Indeed, the developer uses
Django-Rules by setting it as one of the project's ``AUTHENTICATION_BACKENDS``.

In order to use these "predicates" (Django-Rules's name for these dynamic
conditions), they first must be registered with a ruleset and then–at least in
their documented usage pattern–registered as permissions with Django-Rules's
permissions backend. After that, they can simply be used just like permissions
in Django's auth package.

While the approach of treating "rules" as permissions is undeniably attractive,
it is, however, extremely inconsistent with the existing concept of
"permissions" in Django. Arguably, this is how Django should have done it in
the first place, however, given the current situation this would be an
extremely disruptive change. As a third-party package, they can do whatever
they want and it seems to work fine for them. However, if this were added to
Django core it would be very disruptive, confusing, and inconsistent.

Specification
=============

As described in the `Abstract`_, I propose a small new API tentatively named
'Conditions'. Like Django's Forms API and class-based Views, Conditions would
often be related to particular models and/or views, but would still be written
without any special knowledge of or relationship to how they will be used.

From the developer's end, it would be used as follows:

MyApp/conditions.py::

        from django.contrib import conditions


        class OwnsText(conditions.Condition):
                def evaluate(self, user, obj):
                        return obj.owner == user

        class CanEditText(conditions.UserPermissionCondition):
                permission = 'translations.change_text'

MyApp/views.py::

        from django.views import generic
        from django.contrib.conditions import mixins
        from MyApp import conditions
        from MyApp.models import Text


        class EditTextView(mixins.RequiredConditionsMixin, generic.UpdateView):
                model = Text
                execute_conditions = (conditions.CanEditText, conditions.OwnsText)

As should be clear from the above code, a user attempting to access the above
view would have to be listed as the owner of the Text in question (as
represented by the value of its ``.owner`` ForeignKey), and be assigned the
``'translations.change_text'`` permission according to auth.  Otherwise, (if
the following behavior is not overridden by the developer) it will raise a
``PermissionDenied`` error with an appropriate message provided automatically
by a method ``conditions.UserObjectCondition`` or
``conditions.UserPermissionCondition``, or both, if they both failed.

Raising ``PermissionDenied`` is, of course, a security issue in certain cases.
Therefore, when appropriate, conditions required to perform an action will be
divided into 'access' conditions and 'execute' conditions. The 'access'
conditions must be passed first or else the attempted action will result in a
mere "Page not found" error.

Implementation
==============

The Core
--------

ConditionResults
~~~~~~~~~~~~~~~~

``ConditionResult`` would be the most straightforward concept and piece of code
introduced in the API.  Simply put, it would be a data structure used to
convey:

- A boolean value of ``True`` or ``False`` to indicate a pass or fail,
- A message in a string in the case of a fail,
- A link to the condition that produced it, and
- The keyword arguments with which it was run.
  
It's implementation would be approximately as follows::

        class ConditionResult:
                def __init__(self, condition, passed, kwargs, message=None):
                        self.condition = condition
                        self.passed = passed
                        self.message = message
                        self.kwargs = kwargs

                def __bool__(self):
                        return self.passed

                def __str__(self):
                        return self.message

BaseCondition
~~~~~~~~~~~~~

Anything that is a Condition or acts like it would be a sub-class of one
(essentially abstract) superclass: BaseCondition. BaseCondition's basic
structure (and therefore the structure of all Conditions) would be roughly as
follows::

        class BaseCondition:
                message = ''

                def get_message(self, **kwargs):
                        if message:
                                return message
                        raise NotImlementedError()

                def check(self, **kwargs):
                        return NotImplementedError()

Condition
~~~~~~~~~

In order to simplify the process of custom Conditions, the developer is
encouraged to write their Conditions as subclasses of ``Condition``, itself a
subclass of ``BaseCondition``. It simplifies the work of the developer; instead
of writing a method that returns a ``ConditionResult`` filled with the
appropriate message, the developer only needs to provide an ``evaluate()`` that
returns a boolean, and a proper definition of ``get_message()`` if appropriate.

Draft implementation::

        class Condition(BaseCondition):
                def evaluate(self, **kwargs):
                        raise NotImplementedError()

                def check(self, **kwargs):
                        relevant_kwargs = {}
                        inspection = inspect.getargspec(self.evaluate)
                        if inspection.keywords:
                                relevant_kwargs = kwargs
                        else:
                                for kwarg in kwargs:
                                        if kwarg in inspection.args:
                                                relevant_kwargs[kwarg] = kwargs[kwarg]
                        result = self.evaluate(**relevant_kwargs)
                        if result:
                                return ConditionResult(passed=True, condition=self, kwargs=kwargs)
                        return ConditionResult(passed=False, message=self.get_message(**kwargs_to_check), condition=self, kwargs=kwargs)

In other words, ``check()`` provides a hook for the invoker of the condition to
run the condition and return a ``ConditionResult``, by passing the keyword
arguments necessary for the condition to evaluate (which, in the vast majority
of cases, would be either ``user`` or both ``user`` and ``object``).
``evaluate()``, on the other hand, is a hook for the developer (or provided
subclasses for common usage cases) to override in order to define the
meaningful logic of the condition.

ConditionCombiners
~~~~~~~~~~~~~~~~~~

There would also of course be Conditions that simply combine other Conditions
as if with a boolean operator. The two "combiners" provided by this proposal
would be ``EveryCondition`` and ``AnyCondition``. They would each be
sub-classes of ``BaseCondition`` and would act just like ordinary Conditions.
Their ``evaluate()`` would go through a given iterable of Conditions,
``check()``-ing each one the appropriate kwargs. The ``.message`` of the
CondtionResults that they return would by default be a concatenation of all of
the results of the ``.message``'s of the results of said call of ``check()``.

``EveryCondition`` would only return ``True`` if all of its member Conditions
return ``True``, while ``AnyCondition`` would return ``True`` if any of its
member Conditions return ``True``.  The Condition combiners would of course be
nestable.

Draft implementation::

        class ConditionCombiner(BaseCondition):
                conditions = ()
                booelan_func = None

                def check(self, **kwargs):
                        results = []
                        for condition in self.conditions:
                                results.append(condition().check(**kwargs))
                        return ConditionResult(condition=self, message=self.get_message(results), passed=boolean_func(results), kwargs=kwargs)

                def get_message(self, results): # TODO: Does it really make sense to override and call get_message()?
                        joiner.join([result.message for result in results])


        class EveryCondition(BaseCondition):
                boolean_func = all
                joiner = '\nAND\n'


        class AnyCondition(BaseCondition):
                boolean_func = any
                joiner = '\nOR\n'

Notice that ``ConditionCombiner`` —and therefore its sub-classes—does *not*
override ``evaluate()`` as most developer-made (i.e. custom) Conditions would,
due to the fact that the logic of ``get_message()`` is dependent on the value
of ``.message`` in the ConditionResult's returned by the evaluation logic of
the Condition Combiner.

The Hooks
---------

Helper Function
~~~~~~~~~~~~~~~

As for function-based views, since Conditions are essentially just fancy
functions, developers could easily write their own code that utilizes their
conditions. The API, however, would provide a helper function that would run
the given Condition(s) and handle the Auth-related issues (raise
``PermissionDenied``, etc.) on failure. It would also allow the developer to
provide callback functions to modify default behavior.

Django-Rules's technique of using a decorator presents issues when the
function-based view at hand gets an object (e.g. a Model instance from the
ORM), as this object is not accessible to the decorator. Django-rules solved
this by allowing the developer to provide a function (as a callback) that
returns the necessary object.

This causes its own problems, though. First, a model instance will have to be
retrieved from the database twice–an unacceptable performance cost. Second–and
more importantly–it forces the developer to twice define their logic for
retrieving that object. An experienced developer can mitigate some of the
issues that this pattern raises by having both the in-view logic and the
permissions-related callback both refer to a third function to get the job
done. However, this adds unnecesssary complication and is not prescribed by the
documentation.

Given these drawbacks, this proposal would instead bring in two options for
developers to use for authorization in their function-based views:

1. ``check_conditions()``, a function for the developer to call in their
   function-based views with arguments ``access``, ``execute``, and ``kwargs``.
   It would pass the arguments defined in dictionary ``kwargs`` to the
   Conditions listed in tuples ``access`` and subsequently in ``execute``.
   Should any Condition in ``access`` return ``False``, the function raises an
   ``Http404`` exception. If all of those pass, and then any Condition in
   ``execute`` fails, it will raise a ``PermissionDenied``.

   Reference implementation::

        from django.http import Http404
        from django.core.exceptions import PermissionDenied
        from django.contrib.conditions.combiners import EveryCondition

        def default_no_access_handler():
                raise Http404


        def default_no_execute_handler(message):
                raise PermissionDenied(message)


        def check_conditions(condition_kwargs, access, execute, no_access_handler=default_no_access_handler, no_execute_handler=default_no_execute_handler):
                for condition in access:
                        if not condition.run(**condition_kwargs):
                                no_access_handler()
                results = []
                execute_conditions = EveryCondition(conditions=execute)
                execute_conditions_result = execute_conditions.check(**condition_kwargs)
                if not execute_conditions_result.passed:
                        no_execute_handler(message=execute_conditions_result.message)

2. ``@condition_protected``, a decorator whose functionality is primarily
   achieved by calling the above-described function.  It determines the value
   of ``kwargs`` by a developer-defined function provided as an argument for
   the parameter ``get_kwargs``. Unlike in Django-Rules's version, however, the
   result of ``get_kwargs`` is then passed to the wrapped function-based view.
   Through this pattern, the object needn't be retrieved from the database
   twice, and the problems that could arise from a technique involving
   duplication of logic are mitigated because there is no duplication.

    It would be used as in the following example::

        from django.contrib.conditions.decorators import check_conditions
        from django.shortcuts import get_object_object_or_404
        from Bookclubs.models import *


        def get_club(request, pk):
                return {'user': request.user, 'club': get_object_object_or_404(Club, pk=pk)}


        @condition_protected(access=(IsAuthenticated,), execute=(InClub,), get_kwargs=get_club)
        def club_detail(request, club):
                pass

Reference Implementation::

        import inspect
        from django.contrib.conditions import shortcuts


        def condition_protected(view, access, execute, get_kwargs):
                def wrapped_view(request, *args, **kwargs):
                        condition_kwargs = get_kwargs(request, *args, **kwargs)
                        shortcuts.check_conditions(condition_kwargs, access, execute)
                        inspection = inspect.getargspec(view)
                        if inspection.keywords:
                                kwargs.update(condition_kwargs, *args, **kwargs)
                        else:
                                for kwarg in condition_kwargs.keys():
                                        if kwarg in inspection.args:
                                                kwargs[kwarg] = condition_kwargs[kwarg]
                        return view(request, *args, **kwargs)
                return wrapped_view

Class-based View Mixins
~~~~~~~~~~~~~~~~~~~~~~~

The next tie-in/hook to the core of the Conditions API would be mixins for the
Django's generic class-based views. There would be multiple different mixins to
be mixed-in variously depending on whether the class-based view its being mixed
into has a ``get_object()`` method (that actually gets called) or not. The
developer would provide the Conditions they want checked in two tuples,
``access_conditions`` and ``execute_conditions``. If any Condition in
``access_conditions`` fails, the view would by default return a 404 (page not
found). If those pass, but a Condition in ``execute_conditions`` fails, the
view would by default return a 403 (permission denied).

In order to reduce the amount of research and trial-and-error required of
developers, the API would provide special sub-classes of the generic views with
the appropriate mixin already mixed in.

Exactly what happens when the Conditions fail could be dictated by the
developer by overriding the ``no_access()`` and ``no_execute()`` methods, whose
default behaviors, respectively, would be to raise ``Http404`` or
``PermissionDenied`` with the appropriate message.

Draft implementation::

        from django.conf import settings
        from django.views.generic.detail import SingleObjectMixin


        class ConditionsMixin:
                access_conditions = ()
                execute_conditions = ()

                def get_condition_kwargs(self, request, *args, **kwargs):
                        condition_kwargs = {'request': request}
                        if 'django.contrib.sessions.middleware.SessionMiddleware' in settings.MIDDLEWARE_CLASSES:
                                condition_kwargs['session'] = request.session
                                if 'django.contrib.auth.middleware.AuthenticationMiddleware' in settings.MIDDLEWARE_CLASSES:
                                        condition_kwargs['user'] = request.user
                        return condition_kwargs

                def no_access(self):
                        raise Http404

                def no_execute(self, results):
                        raise PermissionDenied(

                def can_check_object_conditions(self):
                        return issubclass(self.__class__, SingleObjectMixin)

                def dispatch(self, request, *args, **kwargs):
                        if not self.can_check_object_conditions():
                                condition_kwargs = self.get_condition_kwargs(request)
                                check_conditions(condition_kwargs, self.access_conditions, self.execute_conditions, no_access_handler=self.no_access, no_execute_handler=self.no_execute)
                        return super(ConditionsMixin, self).dispatch(request, *args, **kwargs)

                def get_object(self, *args, **kwargs):
                        obj = super(ConditionsMixin, self).get_object(*args, **kwargs)
                        if self.can_check_object_conditions():
                                condition_kwargs = self.get_condition_kwargs(request)
                                condition_kwargs['obj'] = obj
                                check_conditions(condition_kwargs, self.access_conditions, self.execute_conditions, no_access_handler=self.no_access, no_execute_handler=self.no_execute)
                        return obj

Template Tags
~~~~~~~~~~~~~

The project would include template tags that would allow the developer to test
their conditions within their templates. Like the view mixin or the view
decorator, the tag would accept a series of Conditions, which would then all be
tested. The keyword arguments that Conditions must be given in order to pass
may be provided as keyword arguments in the tag. In all practical cases, the
developer would utilize the result by storing it in a template context variable
using the ``as`` keyword.

In order for these Conditions (the Condtions themselves, not their results) to
be accessible within the template context, Conditions could be passed to the
template context on a per-view basis. However, because this technique
necessarily increases coupling of views and templates, this project will
include and through documentation prescribe the use of a template context
processor. Said context processor will work by going through each member of
``INSTALLED_APPS``, and from each one's ``conditions.py`` module including
every object that is a subtype of ``BaseCondition`` in a dictionary
``conditions``.
         
Example usage::

        {% load conditions %}

        {% check_conditions conditions.OwnsText conditions.CanEditText user=user request=request obj=text as can_edit %}
        {% if can_edit %}
                <a href="{% url 'texts:text.edit' pk=text.pk %}">{% trans 'Edit Text' %}</a>
        {% endif %}

Draft implementation:

Template Tag::

        from django import template


        register = template.Library()


        @register.simple_tag
        def check_conditions(*conditions, **kwargs):
                # TODO: What to do in case of no conditions?
                for condition in conditions:
                        result = condition.check(**kwargs)
                        if not result:
                                return False
                return True

Context Processor::

        def conditions(request):
                

Form Choices Filter
~~~~~~~~~~~~~~~~~~~

The API would include another mixin that would be mixed into ``FormView`` (and
its sub-classes) that would narrow all the members of the ``.queryset``'s of
relational fields to ones that match a given Condition. This would be used, for
example, on a ``CreateView``, where the developer wants to limit the user to
viewing and selecting instances of which they are the owner (as determined by a
``ForeignKey``).

However, running a Condition against every instance in a queryset can quickly
become very inefficient. For cases when it would be necessary, the mixin would
provide a callback to allow the developer to use whatever means they want to
more efficiently narrow down the queryset before the Conditions are run against
its instances. This may seem like redundant code, however the purposes of the
two different "narrowing" methods are not the same, one is for efficiency, one
is for security.

Rough implementation::

        class FormChoicesConditionMixin:
                field_choices_conditions = {} # ex: {'reader': (MemberInLeaderClub,)}

                def get_form(self, *args, **kwargs):
                        form = super(FormChoicesConditionMixin, self).get_form(*args, **kwargs)
                        for field, conditions in self.field_choices_conditions.items():
                                narrowed_queryset = self.narrow_queryset(field, form.fields[field].queryset) # pre-narrows the queryset for efficiency
                                condition_queryset = generate_condition_queryset( # function that narrows down queryset to just members that pass conditions
                                        queryset=narrowed_queryset,
                                        conditions=conditions
                                )
                                form.fields[field].queryset = condition_queryset
                        return form

                def form_valid(self, form, *args, **kwargs):
                        for field, conditions in self.field_choices_conditions.items():
                                check_conditions(form.cleaned_data[field], conditions) # function that checks values against conditions and raises exceptions accordingly
                        return super(FormChoicesConditionMixin, self).form_valid(form, *args, **kwargs)

                def narrow_queryset(self, field, queryset):
                        '''To be overridden by the developer, should efficiently return a
                        narrowed down queryset (not necessarily a completely secure one)
                        for field <field> given <queryset>
                        '''
                        return queryset

Rationale
=========

An object-oriented design standard for the Conditions themselves (rather than a
function-based one) was selected in order for the API to provide extendable
default Conditions for common usage cases (e.g. permissions-based), as well as
extendable common behavior (such as the message provided on failure).

The reason for dividing conditions into "access" and "execute" conditions is to
not give an attacker any information that they shouldn't have access to (e.g.
whether an instance with a particular PK exists).

Undecided
=========

The following items are under consideration for inclusion with the proposal:

QuerySet-Based Conditions
-------------------------

This proposal could potentially include a common-usage case subclass of ``Condition`` called ``QuerySetCondition``.

Example implementation::

        class QuerySetCondition(BaseCondition):
                queryset = None
                inclusive = True

                def get_queryset(self, **kwargs):
                        if self.queryset:
                                return self.queryset
                        raise NotImplementedError()

                def evaluate(self, obj, **kwargs):
                        return obj in self.get_queryset(**kwargs)

Reasons against:

- Reduces orthogonality; QuerySet-based Conditions would be dependent on the
  Django ORM where no other Condition was.
- Inefficient evaluation; evaluating a Condition written like this would
  requiring actually querying the database and checking that a given instance
  is in said queryset, which could take significantly more time than merely
  evaluating an attribute of a Python object.
- Technically there is no guarantee that an object that evaluates to ``True``
  is actually a member of the result of ``get_queryset()`` in a developer's
  subclass of ``QuerySetCondition`` (or the opposite). One of those depends on
  the result of ``evaluate()``, which could easily be overridden, while the
  other depends on the result of ``get_queryset()``.

Copyright
=========

This document has been placed in the public domain per the Creative Commons
CC0 1.0 Universal license (https://creativecommons.org/publicdomain/zero/1.0/deed).
