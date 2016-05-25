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

A few other packages have attempted to solve this problem. Below are outlined
their methods, as well as their key differences with this proposal.

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
cleanly separates action from rule throughout the framework.

Django Rest Framework's permissions' encapsulation is further limited in that
their rule-checking method must take ``view`` as argument, thus limiting each
Permission's usability to authenticating in the context of a view (as opposed
to, say, in a template to determine whether a link should be displayed).

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
permissions backend.

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
'Conditions'. Analogous to Django's Forms API and class-based Views, Conditions
would often be related to particular models and/or views, but would still be
written without any special knowledge of or relationship to how they will be
used.

From the developer's end, this would work as follows:

MyApp/conditions.py::

        from django.contrib import conditions
        # other imports


        class OwnsText(conditions.UserObjectCondition):
                def evaluate(self, user, object):
                        return object.owner == user

        class CanEditText(conditions.UserPermissionCondition):
                permissions = ('translations.change_text',)

MyApp/views.py::

        from django.views import generic
        from django.contrib.conditions import mixins
        from MyApp import models
        from MyApp import conditions


        class EditTextView(mixins.RequiredConditionsMixin, generic.UpdateView):
                model = models.Text
                required_conditions = (conditions.CanEditText, conditions.OwnsText)

As is probably fairly clear from the above code, a user attempting to access
the above view would have to be listed as the owner of the Text in question (as
represented by the value of its ``.owner`` ForeignKey), and be assigned the
``'translations.change_text'`` permission according to auth.  Otherwise, (if
the following behavior is not overridden by the developer) it will raise a
``PermissionDenied`` error with an appropriate message provided automatically
by a method ``conditions.UserObjectCondition`` or
``conditions.UserPermissionCondition``, or both, if they both failed.

*Raising ``PermissionDenied`` is, of course, a security issue in certain cases.
Therefore, a way of producing mere 404 errors when appropriate is detailed later
in this proposal.*

Rationale
=========

An object-oriented design standard for the Conditions themselves (rather than a
function-based one) was selected in order for the API to provide
easily-extendable default Conditions for common usage cases (e.g.
permissions-based or 

#. Rationale -- The rationale fleshes out the specification by describing what
   motivated the design and why particular design decisions were made.  It
   should describe alternate designs that were considered and related work.

   The rationale should provide evidence of consensus within the community and
   discuss important objections or concerns raised during discussion.

#. Reference Implementation -- The reference implementation must be completed
   before any DEP is given status "Final", but it need not be completed before
   the DEP is accepted.  While there is merit to the approach of reaching
   consensus on the specification and rationale before writing code, the
   principle of "rough consensus and running code" is still useful when it comes
   to resolving many discussions of API details.

   The final implementation must include tests and documentation, per Django's
   `contribution guidelines <https://docs.djangoproject.com/en/dev/internals/contributing/>`_.

Draft Implementation
====================

The Implementation Part One - The Core
--------------------------------------

ConditionResults
~~~~~~~~~~~~~~~~

``ConditionResult`` would be the simplest concept/code introduced in the API.
Simply put, it would be a data structure used to convey:

- A boolean value of ``True`` or ``False`` to indicate a pass or fail,
- A message in a string in the case of a fail,
- A link to the condition that produced it, and
- The keyword arguments with which it was run.
  
It's implementation would look something like this::

        class ConditionResult:
                def __init__(self, condition, passed, message=None, kwargs):
                        self.condition = condition
                        self.passed = passed
                        self.message = message
                        self.kwargs = kwargs

                def __bool__(self):
                        return self.passed

                def __str__(self):
                        return self.message

Conditions
~~~~~~~~~~~~~~~~~~~~

Conditions would all be sub-classes of one super-class, BaseCondition.
BaseCondition's basic structure would be roughly as follows::

        class BaseCondition:
                message = ''
                kwargs = None

                def get_message(self, **kwargs):
                        if message:
                                return message
                        raise NotImlementedError()

                def evaluate(self, **kwargs):
                        raise NotImplementedError()

                def check_kwargs(self, kwargs):
                        missing_kwargs = []
                        for kwarg in self.kwargs:
                                if kwarg not in kwargs:
                                        missing_kwargs += [kwarg]
                        if missing_kwargs:
                                raise ValueError('Missing keyword arguments: %s' % str(missing_kwargs))

                def run(self, **kwargs):
                        self.check_kwargs(kwargs)
                        result = self.evaluate(**kwargs)
                        if result:
                                return ConditionResult(passed=True, condition=self, kwargs=kwargs)
                        return ConditionResult(passed=False, message=self.get_massage(**kwargs_to_check), condition=self, kwargs=kwargs)

Put simply, ``run()`` provides a hook for the invoker of the condition to,
well, run the condition, by passing the keyword arguments necessary for the
condition to evaluate (which, in the vast majority of cases, would be either
``user`` or both ``user`` and ``object``). ``evaluate()``, on the other hand,
would be a hook for the developer (or sub-classes for common usage cases) to
override in order to define the meaningful logic of the condition.

Combiners
~~~~~~~~~~~

There would also of course be classes for combining multiple conditions into
one. The two "combiners" would be ``EveryCondition`` and ``AnyCondition``. They
would each be sub-classes of ``BaseCondition`` and would act just like ordinary
Conditions. Their ``evaluate()`` would go through a given iterable of
Conditions, ``run()``-ing each one the appropriate kwargs. Their default
``get_message()`` would return a concatenation of all of the results of the
``.message``'s of the results of said ``run()``-ing.

``EveryCondition`` would only return ``True`` if all of its member Conditions
return ``True``, while ``AnyCondition`` would return ``True`` if any of its
member Conditions return ``True``.  The Condition combiners would of course be
nestable.

Implementation Part Two - The Hooks
-----------------------------------

Class-based View Mixins
~~~~~~~~~~~~~~~~~~~~~~~

The first tie-in/hook to the core of the Conditions API would be mixins for the
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
developer by overriding the ``condition_fail()`` method, whose default behavior
would be cannibalized from Django's own ``AuthMixin`` and could also be
customized by modifying attributes of the view.

Helper Function
~~~~~~~~~~~~~~~

As for function-based views, since Conditions are essentially just fancy
functions, developers could easily write their own logic based on their
conditions. The API, however, *would* provide a helper function that would
run the given Condition(s) and handle the Auth-related issues (redirect to
login, etc.) on failure. It would also allow the developer to provide callback
functions to modify default behavior.

Django-Rules's technique of using a decorator presents issues when the
function-based view at hand gets an object (e.g. a Model instance from the
ORM), as this object is not accessible to the decorator. Django-rules has
overcome this by allowing the developer to provide a function (as a callback)
that returns the necessary object.

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

2. ``@check_conditions``, a decorator (same name, different module) whose
   functionality is primarily achieved by calling the above-described function.
   It determines the value of ``kwargs`` by a developer-defined function
   provided through as an argument for the parameter ``get_kwargs``. Unlike in
   Django-Rules's version, however, the result of ``get_kwargs`` is then passed
   to the wrapped function-based view. Through this pattern, the object needn't
   be retrieved from the database twice, and the problems that could arise from
   a technique involving duplication of logic are mitigated because there is no
   duplication.

::
        from django.contrib.conditions.decorators import check_conditions
        from django.shortcuts import get_object_object_or_404
        from Bookclubs.models import *


        def get_club(request, pk):
                return {'user': request.user, 'club': get_object_object_or_404(Club, pk=pk)}


        @check_conditions(access=(IsAuthenticated,), execute=(InClub,), get_kwargs=get_club)
        def club_detail(request, club):
                pass

   Some idea of how the decorator would be implemented, in decorators.py::
        import inspect
        from django.contrib.conditions import shortcuts


        def check_conditions(view, access, execute, get_kwargs):
                def wrapped_view(request, *args, **kwargs):
                        condition_kwargs = get_kwargs(request, *args, **kwargs)
                        kwargs.update(condition_kwargs, *args, **kwargs)
                        inspection = inspect.getargspec(view)
                        return view(request, *args, **kwargs)
                return wrapped_view

ModelAdmin Mixin
~~~~~~~~~~~~~~~~

This would admittedly be the least-useful hook of the bunch, as ``ModelAdmin``
itself is thankfully already very easy to extend with dynamic logic to limit
access. Still, the mixin provided by this project would at least allow
developers to neatly organize their Conditions into tuples stored in a class
attribute.

**RESOLVE**: Should this mixin default to also requiring the proper
Django model permissions, as the vanilla ModelAdmin does?)

Pass to Template Context
~~~~~~~~~~~~~~~~~~~~~~~~

The API would add another mixin and equivalent helper function, which would run
a given tuple of Conditions and pass the results to the template context. The
template writer can then use the results of these Conditions to filter what the
user sees or determine whether to show a link based on whether it would be
accessible to the current user.

*Note*:

The original specification concept was for the developer to list inside a tuple
attribute of the View class which conditions should have their results passed
to the template context.  However, some very strong arguments have been made
for giving Template writers access to all conditions in the form of tags. This
solution, on the other hand, could cause some very unpredictable namespace
conflicts. Community discussion on this has only just begun, and with luck
should arrive on a solution in the months befor GSoC begins.

Form Choices Filter
~~~~~~~~~~~~~~~~~~~

The API would include another hook mixin that would be mixed into FormView (and
its sub-classes) that would narrow all the members of the ``.queryset``'s of
relational fields to ones that match a given Condition. This would be used, for
example, on a CreateView, where the developer wants to limit the user to
viewing and selecting instances of which they are the owner (as determined by a
ForeignKey).

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


Copyright
=========

This document has been placed in the public domain per the Creative Commons
CC0 1.0 Universal license (https://creativecommons.org/publicdomain/zero/1.0/deed).
