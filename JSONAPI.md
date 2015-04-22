# The API described by jsonapi.org

Which I will refer to as 'jsonapi.orgese'.

And why I don't think it's worth bothering with.

- We already have a convention (I'll call it Obvious JSON) that's more
  intuitive and easier to implement
  - Obvious serialization of objects as JSON with conventions for the less-obvious parts
  - Related objects can be embedded in each other
    e.g. a house might have residents, so you'd say ```{"residents": [{"name": "Bob"}, {"name":"Sally"}]}```
  - No 'links', 'linkages', 'self', 'related' needed.
    - Can you even tell me what those things /mean/ in jsonapi.orgese?
- JSONAPI.orgese uselessly combines data (the stuff you normally stuff in a JSON document)
  with metadata (that's all the links stuff).
  - A human would be able to make sense of the data without the extra metadata.
  - A machine isn't going to be able to do anything useful with the metadata anyway.
  - The combination adds unnecessary complexity to the task of interpreting the data.
  - The metadata doesn't belong in there.
- JSONAPI.orgese is an additional language that sits between JSON and your application
  - Even the designers recognize this; they defined its own MIME type
  - This layer adds complexity to the application and makes API
    requests harder to understand but doesn't buy us anything
- How do you adapt JSONAPI.org-formatted data to non-request-response situations?
  - You have to come up with your own conventions for these cases anyway
  - You'll probably end up using Obvious JSON for this part, anyway, so why not just use it throughout?
- Following somebody else's standard for the sake of following a standard doesn't buy us anything.
- JSONAPI.orgese isn't even a standard, yet.  They're on 'release candidate 3'.
  - The reason it's taking them so long to make an official release is because it's overcomplicated.
  - The last thing we should be doing is standardizing on a convention that's still changing.
- JSONAPI.org leaves some important bits unspecified
  - e.g. filtering ('Note: JSON API is agnostic about the strategies
    supported by a server. The filter query parameter can be used as
    the basis for any number of filtering strategies.'
    -- [http://jsonapi.org/format/#fetching-filtering])
  - e.g. paging

In summary, jsonapi.org defines a bunch of stuff to make your JSON
harder to deal with, but doesn't specify enough to make a useful
standard for many parts of an API that our applications rely on.