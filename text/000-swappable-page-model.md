# RFC 000: Swappable Page model

* RFC: 000
* Author: Sonny Baker
* Created: 2020-06-19
* Last Modified: 2020-06-19

## Abstract

This RFC proposes that Wagtail should give users the option to specify a custom `Page` model,
rather than inheriting the concrete one provided at `wagtail.core.models.Page`.

## Specification

Django has undocumented support for [swappable models](https://code.djangoproject.com/ticket/19103),
most commonly used when specifying a custom `auth.User` model.
[A relatively simple third party library](https://github.com/wq/django-swappable-models) provides an
API for leveraging this feature.

Wagtail core would provide an abstract version of the existing `Page` model
(with default attributes, methods, object manager, queryset etc), and helper methods to retrieve the
correct page model, falling back to a default:

```python
class AbstractPage(MP_Node, index.Indexed, ClusterableModel, metaclass=PageBase):

    objects = PageManager()

    title = models.CharField(
        verbose_name=_('title'),
        max_length=255,
        help_text=_("The page title as you'd like it to be seen by the public")
    )

    # ...

    class Meta:
        swappable = swapper.swappable_setting('wagtailcore', 'Page')

class Page(AbstractPage):

    class Meta(AbstractPage.Meta):
        pass


def get_page_model():
    """
    Get the site model from the ``WAGTAILCORE_PAGE_MODEL`` setting.
    Defaults to the standard :class:`~wagtail.sites.models.Page` model
    if no custom model is defined.
    """
    Page = swapper.load_model("wagtailcore", "Page")
    return Page


def get_page_model_string():
    """
    Get the dotted ``app.Model`` name for the page model as a string.
    """
    return getattr(settings, 'WAGTAILCORE_PAGE_MODEL', 'wagtailcore.Page')

```

This would be inherited by the user's custom `Page` class:

```python
from wagtail.core.models import AbstractPage

class Page(AbstractPage):
    thumbnail = models.ForeignKey(
        get_image_model_string(),
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='+'
    )

    class Meta(AbstractPage.Meta):
        pass
```

And the project `settings.py` file will point to the dot notated string representation of that model:

```python
WAGTAILCORE_PAGE_MODEL = 'foo.Page'
```

This would mean future references to the page model can be made like so:

```python
class FooPage(Page):
    related_page = models.ForeignKey(
        get_page_model_string(),
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='+'
    )
```

### Rationale
* Improves performance - fewer lookups as shared page attributes will be on the concrete `Page` model
* Reduces code complexity - filtering by custom attributes like `Page.objects.filter(language='fr')`
* Removing `.specific()` from queries will allow annotations of `Page` querysets
* Allows developers to control routing and URL generation
* Consistent with custom `Image` and `Document` (and [potentially `Site`](https://github.com/wagtail/wagtail/pull/5457)) models
* Simplifies managing future extensions with mixins - e.g Translation, Experiments, Personalisation

### Caveats / Considerations
* This is a major change to Wagtail, and must be tested thoroughly. It would be be best soft-launched
as an experimental feature, to collect user feedback.
* Swappable models is an _intentionally undocumented_ (or "stealth alpha") feature -
[User models were the pilot program for this](https://code.djangoproject.com/ticket/19103). Though its
use in the `auth` library has shown it to be stable, relying on a private API is still a risk, and another solution may be preferable.
* Would mean updating core migrations.
* Page revisions will need to be adapted to work with generic foreign keys.
* Could present issues when adding (or removing) fields from the abstract Page base (ie. clashing attribute names).
* Potentially adds a layer of upfront complexity for new users. The existing custom `Image` and
`Document` model implementations are good examples of optional features available to those who need it,
but ignorable for common Wagtail installations.
* Could make future support queries harder to deal with.
* Difficult to switch to a custom model mid-project. Documentation should specify this in the same
manner as Django does with [custom user models](https://docs.djangoproject.com/en/3.0/topics/auth/customizing/#changing-to-a-custom-user-model-mid-project).

## Open Questions
* How do we implement this in a way that is frictionless for new starters?
* What are the implications for third-party packages?
* What other consequences might there be for introducing this change?

## References / further reading
* [Open PR for swappable `Site` model](https://github.com/wagtail/wagtail/pull/5457/)
* [Open feature request](https://github.com/wagtail/wagtail/issues/3282)
* [Open GitHub issue](https://github.com/wagtail/wagtail/issues/836)
* [Google groups discussion](https://groups.google.com/forum/#!msg/wagtail/4459qj1tNiU/D91COykMSmcJ)
