# Trek the World Wide Web with Treq - THREADS First Take

Most who dabble in Python love the ``requests`` module. It's a great product and can really alleviate the stress when dealing with making HTTP requests. However, sooner rather than later, a requirement will come along that requires multiple requests to be made concurrently. In fact, one of the most frequent questions asked regarding ``requests`` is how to accomplish such concurrency. A rudimentary approach would be something like:


``` python
import threading
try:
    from queue import Queue # 3.X
except:
    from Queue import Queue # 2.X

import requests

def getContent(storage, *urls):
    """
    Take a list of URLs and retrieve the content.

    :return: List of content (str)
    """
    for url in urls:
        thrd = threading.Thread(target=storeContent, kwargs={'url': url, 'q': storage})
        thrd.start()

def storeContent(url, q):
    req = requests.get(url)
    q.put(req.content)
    thread_id = threading.current_thread().name
    print('ThreadID-{0}: Stored content for {1} in the queue.'.format(thread_id, url))

def main():
    storage = Queue()
    urls = [
        'http://swapi.co/api/films/schema',
        'http://swapi.co/api/people/schema',
        'http://swapi.co/api/planets/schema',
        'http://swapi.co/api/species/schema',
        'http://swapi.co/api/starships/schema',
        'http://swapi.co/api/vehicles/schema']

    getContent(storage, *urls)
    counter = 0
    while counter < len(urls):
        content = storage.get()
        # print(content)
        counter += 1

main()
```


This works great especially if all that needs to be done is gather content from a HTTP request. The requests are performed in a thread and the content is stored in a ``Queue``, which can span multiple threads. However, real life isn't that simple and customers rarely only want to do a simple task. More often than not, there will be a need to get content from multiple sites AND perform multiple other tasks in parallel. What's the solution? More threads? Maybe, but there are alternative solutions which have familiar syntax to ``requests``. One such solution is ``treq`` which leverages the asynchronous Twisted framework. This is how a ``treq`` equivalent will look like:


``` python
from twisted.internet import defer, task, reactor
import treq

async def getContent(*urls):
    deferred_list = []
    for url in urls:
        dfr = treq.get(url)
        dfr.addCallback(treq.content)
        deferred_list.append(dfr)
    results = await defer.DeferredList(deferred_list)

    for status, content in results:
        print(content)

def main(reactor):
    urls = [
        'http://swapi.co/api/films/schema',
        'http://swapi.co/api/people/schema',
        'http://swapi.co/api/planets/schema',
        'http://swapi.co/api/species/schema']
    return defer.ensureDeferred(getContent(*urls))

task.react(main)
```


