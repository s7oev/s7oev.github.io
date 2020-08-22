---
layout: post
title: "My first contribution to an open-source project - PyOData"
date: 2020-08-22
excerpt: "Hi @s7oev , than you for taking the time to report this issue. It's not supported yet but feel free to add it."
---

A project I'm working on at my job requires the Python use of services created in ABAP and exposed as RESTful APIs using OData. The current best way of achieving this is through the developed by SAP Python framework [PyOData](https://github.com/SAP/python-pyodata), also available on GitHub as an open-source project.

The original idea behind developing this tool was for it to be used for test automation of OData services ([source](https://blogs.sap.com/2019/08/13/odata-for-pythonistas/)). So probably because of this the 'inlinecount' functionality was not a top priority. It is about getting the results and their total count at the same request (see [the documentation, point 4.9](https://www.odata.org/documentation/odata-version-2-0/uri-conventions/)). That total count is for all results fitting a certain filter, without caring for parameters that would be used for pagination, such as $top (get the top x results) and $skip (skip the first y results). As I think is obvious from the description, it is much more of a functionality to be used for developing applications communicating with an OData service, rather than it being helpful for automated testing.

I opened an issue asking about the functionality because I needed it for my project. Unsurprisingly (based on the above), I got as a response:

> Hi @s7oev , than you for taking the time to report this issue. It's not supported yet but feel free to add it.

My response was that I'd love to try and contribute, although I can't promise I'll be able to deliver a working functionality. Still, a basic implementation of this actually turned out to be relatively easy to implement. The challenge was to return this to the user without breaking API promises. Because the function where this would happen only returned a list of entitites at that point, returning the total count by using, for example, a dictionary, would break existing code.

After sharing the issue with the package author, he came up with a very useful suggestion. By implementing the Python's List interface, I could add a total_count property, and not break any API promises at the same time. Ultimately, what I ended up doing was almost exactly this, but instead I inherited from the List class:

```
class ListWithTotalCount(list):
    """A list with the additional property total_count"""

    def __init__(self, total_count):
        super(ListWithTotalCount, self).__init__()
        self._total_count = total_count

    @property
    def total_count(self):
        """Count of all entities"""
        if self._total_count is None:
            raise ProgramError('The collection does not include Total Count '
                               'of items because the request was made without '
                               'specifying "count(inline=True)".')

        return self._total_count
```

Other than this, the rest of the productive code was mainly just translating the user request for inlinecount to a URI option ($inlinecount=allpages) and creating the ProgramError exception, which I was asked to do. Writing tests for the functionality actually took most of the time, because I wanted to go for a highly sufficient coverage. Luckily, I also had the tests for $count (return just the count) available, so I had some structure to follow. Here's one example test that I wrote:

```
@responses.activate
def test_inlinecount_with_skip(service):
    """Check getting entities with $inlinecount with $skip"""

    # pylint: disable=redefined-outer-name

    responses.add(
        responses.GET,
        "{0}/Employees?$inlinecount=allpages&$skip=1".format(service.url),
        json={'d': {
            '__count': 3,
            'results': [
                {
                    'ID': 22,
                    'NameFirst': 'John',
                    'NameLast': 'Doe'
                },{
                    'ID': 23,
                    'NameFirst': 'Rob',
                    'NameLast': 'Ickes'
                }
            ]
        }},
        status=200)

    request = service.entity_sets.Employees.get_entities().skip(1).count(inline=True)

    assert isinstance(request, pyodata.v2.service.GetEntitySetRequest)

    assert request.execute().total_count == 3
```

Then, I opened a pull request. I was asked to first provide documentation and changelog, which I also did. And, eventually, this resulted in the pull request being approved. So, my first commit to an open source project is already a reality:

![My commit to PyOData](/img/2020/08/commit-s7oev.png "My commit to PyOData")

Contributing to this project has definitely been a rewarding experience. And, hopefully, it will be just the first of many!
