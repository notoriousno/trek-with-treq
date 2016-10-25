# Trek the World Wide Web with Treq

Most who dabble in Python love the ``requests`` module. It's a great product and can really alleviate the stress when dealing with making HTTP requests. However, sooner rather than later, a requirement will come along that requires multiple requests to be made concurrently. In fact, one of the most frequent questions asked regarding ``requests`` is how to accomplish concurrency. A rudimentary approach would be something like:

``` python
import threading
try:
    from queue import Queue # 3.X                                                                                                                                                                                        except:                                                                                                                                                                                                                      from Queue import Queue # 2.X

import requests

def getContent(*urls):
    """
    Take a list of URLs and retrieve the content.

    :return: List of content (str)
    """
    thread_list = []    # a simple list to store the threads
    storage = Queue()   # a queue to store the content

    for url in urls:
        thrd = threading.Thread(target=storeContent, kwargs={'url': url, 'q': storage})
        thrd.start()
        thread_list.append(thrd)

    content_list = []
    for thrd in thread_list:
        thrd.join()                 # block waiting for the requests to finish
        content = storage.get()     # pops the content from the queue
        content_list.append(content)

    return content_list

def storeContent(url, q):
    req = requests.get(url)
    q.put(req.content)
    thread_id = threading.current_thread().name
    print('ThreadID-{0}: Stored content for {1} in the queue.'.format(thread_id, url))

def main():
    urls = [
        'http://swapi.co/api/films/schema',
        'http://swapi.co/api/people/schema',
        'http://swapi.co/api/planets/schema',
        'http://swapi.co/api/species/schema',
        'http://swapi.co/api/starships/schema',
        'http://swapi.co/api/vehicles/schema']
    content_list = getContent(*urls)
    for x in content_list:
        print(x)

main()
```

This works great especially if all that needs to be done is gather content from a HTTP request. The requests are performed in a thread and the content is stored in a ``Queue``, which can span multiple threads. However, real life isn't that simple and customers rarely only want to do a simple task. More often than not, there will be a need to get content from multiple sites AND perform multiple other tasks in parallel. What's the solution? More threads? Maybe, but there are alternative solutions which have familiar syntax to ``requests``. One such solution is ``treq`` which leverages the asynchronous Twisted framework. This is how a ``treq`` equivalent will look like:

``` python
from twisted.internet import reactor, defer
import treq

import threading

@defer.inlineCallbacks
def getContent(*urls):
    deferred_list = []
    storage = []

    for url in urls:
        deferred = storeContent(url, storage)
        deferred_list.append(deferred)

    results = yield defer.DeferredList(deferred_list)
    defer.returnValue(storage)

def storeContent(url, storage):
    req = treq.get(url)
    req.addCallback(treq.content)
    req.addCallback(storage.append)

    thread_id = threading.current_thread().name
    print('ThreadID-{0}: Stored content for {1} in the queue.'.format(thread_id, url))
    return req

@defer.inlineCallbacks
def main():
    urls = [
        'http://swapi.co/api/films/schema',
        'http://swapi.co/api/people/schema',
        'http://swapi.co/api/planets/schema',
        'http://swapi.co/api/species/schema',
        'http://swapi.co/api/starships/schema',
        'http://swapi.co/api/vehicles/schema']
    content_list = yield getContent(*urls)
    for x in content_list:
        print(x)

    reactor.stop()

main()
reactor.run()
```

"Whoa there cowboy! You said this would be easier!" this is probobly what you're thinking if you got to this point. Actually, you're right this is too complicated but not for the reasons you think. This is a 1:1 conversion from the previous threaded version, however in the context of Twisted, there are many unnecessary steps. Never the less, a few things need explaining and for those new to Twisted, it may not make sense if the example were not equivalent. 
