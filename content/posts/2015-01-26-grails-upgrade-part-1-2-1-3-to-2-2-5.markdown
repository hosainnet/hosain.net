---
layout: post
title: "Grails Upgrade Part 1: 2.1.3 to 2.2.5"
date: 2015-01-26T19:58:08+00:00
---

We have recently upgraded a fairly large web application from 2.1.3 to 2.4.4.

Since it's a big jump, we split the task into multiple grails upgraded so this post will focus on 2.1.3 to 2.2.5.

The best place to start is the [official upgrade documentation page](http://www.grails.org/doc/2.2.x/guide/upgradingFromPreviousVersionsOfGrails.html).

Here are some of the things we've encountered during the upgrade:

* ###“delete() does not work in withNewSession”###

Sessions are now always flushed after withNewSession closure is executed, unless flush mode is manual: [https://jira.grails.org/browse/GRAILS-9739](https://jira.grails.org/browse/GRAILS-9739)

    SomeDomain.withNewSession {
        SomeDomain.list()*.delete()
    }

becomes

    SomeDomain.withNewSession {session ->
        session.flushMode = FlushMode.MANUAL
        SomeDomain.list()*.delete()
    }

* ###explicitly mock domains due to change in 2.1.4###

If your SomeDomain has a relationship to another domain class, you need to add that to the @Mock annotation list as well: [https://jira.grails.org/browse/GRAILS-9637](https://jira.grails.org/browse/GRAILS-9637)

    class ClassA {
        ClassB b
    }

You have to add both classes to the @Mock annotation in your unit test

    @Mock([ClassA, ClassB])

* ###class property is no longer marshalled as json by default###

Cause: [https://jira.grails.org/browse/GRAILS-8572](https://jira.grails.org/browse/GRAILS-8572)

If you had any code prior to ... that looked like this:

    def json = JSON.parse(controller.response.contentAsString)
    jsonResponse.results[0].class == 'SomeClass'

"jsonResponse.results[0].class" is now null
