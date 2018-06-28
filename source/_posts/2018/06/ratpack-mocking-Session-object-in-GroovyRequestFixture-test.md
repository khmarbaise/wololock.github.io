---
title: 'Ratpack: mocking Session object in GroovyRequestFixture test'
date: 2018-06-24 11:59:57
tags:
    - ratpack
    - groovy
    - spock
    - unit tests
categories:
    - Ratpack Cookbook
---

[Ratpack](https://ratpack.io) allows you unit test handlers using [`GroovyRequestFixture`](https://ratpack.io/manual/1.5.4/api/ratpack/groovy/test/handling/GroovyRequestFixture.html) class. 
The good thing about this approach is that it does not require running the whole application and you can
quickly test if the handler does what you expect. However, if you retrieve objects from Raptack's registry you
will run into a problem - registry in this case is empty.

<!-- more -->

In some cases you may find mocking `Session` object useful. Especially if you only want to retrieve specific object
or value from session and do something with it. `GroovyRequestFixture.handle(chain, closure)` gives you an
access to `registry` through the closure passed in the second parameter.

    GroovyRequestFixture.handle(yourHandler) {
        registry { r ->
            r.add(Session, mockSession)
        }
    }

Here we have registered `mockSession` to be injected anytime `Session` instance is being retrieved from the registry.
Keep in mind that mock object does nothing by default (e.g. it return `null` values for methods invocation) so you will
have to "configure" your mock object to return something significant. For instance: 

    Session mockSession = Mock(Session) {
        get('test') >> Promise.value(Optional.of('Lorem ipsum'))
    }
    
will return `Lorem ipsum` value (as a promise of optional) for `session.get('test')`.
    

And here you can find a full example:


```groovy
import groovy.transform.CompileStatic
import ratpack.exec.Promise
import ratpack.groovy.handling.GroovyChainAction
import ratpack.groovy.test.handling.GroovyRequestFixture
import ratpack.http.Status
import ratpack.jackson.internal.DefaultJsonRender
import ratpack.session.Session
import spock.lang.Specification

import static ratpack.jackson.Jackson.json

class MockSessionSpec extends Specification {

    final Session session = Mock(Session) {
        get('test') >> Promise.value(Optional.of('Lorem ipsum'))
    }

    final GroovyChainAction chainAction = new GroovyChainAction() {
        @Override
        @CompileStatic
        void execute() throws Exception {
            get('foo') {
                get(Session).get('test').map { Optional<String> o ->
                    o.orElse(null)
                }.flatMap { value ->
                    Promise.value(value)
                }.then {
                    render(json([message: it]))
                }
            }
        }
    }

    def "should retrieve message from Session object"() {
        given:
        def result = GroovyRequestFixture.handle(chainAction) {
            uri 'foo'
            method 'GET'
            registry { r ->
                r.add(Session, session)
            }
        }

        expect:
        result.status == Status.OK

        and:
        result.rendered(DefaultJsonRender).object == [message: 'Lorem ipsum']
    }
}

```
