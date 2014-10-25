Speaker Notes:

5 minutes at a time

Lekha: brief "what is autoscale", existing testing, real-service backend
Glyph: generally, what's good about fakes? cf. stop mocking start testing
Lekha: first version of mimic, some of the solutions
Glyph: enhancements: plugin system, per-instance state
Lekha: mimic as a repository of known error conditions
Glyph: future enhancements: control of time, error injection
Lekha: call to action: why and how the openstack community should help

# Glyph

# Lekha: first version of mimic, some of the solutions

Mimic was built as a stand-in service for Identity, Compute and Rackspace Load
balancers, the services that autoscale depends upon. Mimic initially
implemented only the subset of each of these endpoints needed by Autoscale.

The essence of Mimic is pretending.  The first thing that you must do to
interact with it is to pretend to authenticate.

Although Mimic does not validate credentials - all authentications will
succeed - as with the real Identity endpoint, Mimic's identity endpoint has a
service catalog which includes URLs for all the services it provides.  A well
behaved OpenStack client will use the service catalog to look up URLs for its
service endpoints. Such a client will only need two pieces of configuration to
begin communicating with the cloud, i.e. credentials and the identity
endpoint. A client written this way will only need to change the Identity
endpoint to be that of Mimic.

When you ask Mimic to create a server, it pretends to create one. This is not
like stubbing with static responses: when Mimic pretends to build a server, it
remembers the information about that server and will tell you about it in the
subsequent requests.

Mimic was originally created to speed things up. So, it was very important
that it be fast both to respond to requests and to have developers setup. So
it was built using in-memory data structures. Mimic has minimal software
dependencies - almost entirely pure Python. No service dependencies and no
configuration. It is entirely self- contained.

Here is a quick demo,

(Demo here: "Look how easy it is to run Mimic...." bootstrap.mp4)


(Demo here: nova python client pretending to create a server, list servers,
delete server, against mimic.)

Here is an example of how we could use Mimic against the OpenStack nova python
client.  We're including the Mimic identity endpoint as the `AUTH_URL`.

We did this with Autoscale, and pointed its tests at a Mimic instance.  This
reduced the test time exponentially. Before Mimic, the functional tests would
take 15 minutes, and now they run in 30 seconds. The system integration tests
would take 3 *hours* or more, cause if one of the servers in the test remained
in the "building" state for fifteen minutes longer than usual, then the tests
ran fifteen minutes slower.  That 3 *hours* (or more) went down to be less than
3 *minutes* (consistently) to complete!

Our dev VMs are configured to run tests against Mimic, so developers get
immediate feedback on the code being written and they can work offline without
having to worry about uptimes of the upstream systems.

Glyph: But Lekha, what about all the error injection stuff I mentioned before?
Does Mimic do that?  How did this set-up test Autoscale's error conditions?

Well Glyph, I am as pleased as I am suprised that you ask that.  Mimic does
simulate errors.

Earlier, when I said Mimic pretends to create a server, that wasn't entirely
true - sometimes Mimic pretends to *not* create a server.  It uses the metadata
provided during the creation of the server. It inspects the metadata, and sets
the state of the server respectively. For example, setting a `metadata` of say
`"server_building": 30`, will make the server stay in building state for 30
seconds. Similarly, servers can be made to go into error state on creation.

Autoscale's purpose is to get the user the right number of servers, even if a
number of attempts to create one failed. We chose to use `metadata` for error
injection so that requests with injected errors could also be run against real
services. For Autoscale, the expected end result is the same number of servers
created, irrespective of the number of failures. But this behavior may also be
useful to many other applications because retrying is a common pattern for
handling errors. 

However, the first implementation of mimic had some flaws, it was fairly
Rackspace specific. It implemented only the services required by autoscale, and
they were all implemented as the core. It ran each service on a different port,
meaning that for N endpoints you would need not just N port numbers, but N
*consecutive* port numbers. It allowed for testing error scenarios, but only
using the metadata. This was not useful for all cases, for example, for a
control panel that does not not allow every UI action to contain metadata.

(TODO: ask someone if Horizon matches this description?)

Mimic also did not implement multiple regions.  It used global variables for
storing all state, which meant that it was hard to add additional endpoints
with different state in the same running mimic instance.

# Glyph

This is the point at which I joined the project.

Mimic had an ambitious vision: to be a one-stop mock for all OpenStack and
Rackspace services that needed fast integration testing.  However, its
architecture at the time severely limited the ability of other teams to use it
or contribute to it.  As Lekha mentioned, it was specific not only to Rackspace
but to Autoscale.

On balance, Mimic was also extremely simple.  It followed the You Aren't Gonna
Need It principle of extreme programming very well, and implemented just the
bare minimum to satisfy its requirements, so there wasn't a whole lot of
terrible code to throw out or much unnecessary complexity to eliminate.

There is, however, a corrolary to YAGNI, which is E(ITO)YAGNI: Eventually, It
Turns Out, You *Are* Going To Need It.  As Mimic grew, other services within
Rackspace wanted to make use of its functionality, and a couple of JSON
response dictionaries in global variables were not going to cut it any more.

So we created a plugin API.

As Lekha mentioned previously, Mimic's Identity endpoint was the top-level
entry point to Mimic as a service; every other URL was available from within
the service catalog.  As we were designing the plugin API, it was clear that
this top-level Identity endpoint needed to be the core part of Mimic, and
plug-ins would each add an entry for themselves to the service catalog.

Each plugin implements the `IAPIMock` interface, which has only two methods:
`catalog_entries` and `resource_for_region`.

`catalog_entries` takes a tenant ID and returns an iterable of `Entry` objects,
each of which is a name and a collection of `Endpoint` objects, each containing
a region, a URI version prefix, and a tenant ID of its own.

Put more simply, APIs have catalog entries for each API type, which in turn
have endpoints for each virtual region they represent.

`resource_for_region` takes the name of a region, a URI prefix - produced by
Mimic core to make URI for each service unique - and a session store where the
API mock may look up state of the resources it pretended to provision for the
respective tenants. `resource_for_region` returns an HTTP resource associated
with the top level of the given region.  This resource then routes this
request to any tenant- specific resources associated with the full URL path.

Mimic uses the Twisted plugin system, which has a very simple model of how
plugins work: you have an abstract interface - some methods and attributes that
you expect of a plugin - and then plugins register themselves by instantiating
a provider of that interface and placing that instance into a module within a
particular namespace package.  In Mimic's case, that interface is `IAPIMock`
and that package is `mimic.plugins`.

Each service can contribute to the service catalog, provide entries and
endpoints, and each endpoint gets added within the URI hierarchy.

# Lekha: mimic as a repository of known error conditions #

Anyone testing a product, will run into unexpected errors. Thats why we test!
But we dont know what we dont know, and cant be prepared for this ahead of
time right!

When we were running the autoscale tests against Compute, we began to see some
one-off errors. Like, when provisioning a server, the
test expected a server to go into a building state for some time before it is
active, __but__ it would remain in building state for over an hour or even
would sometimes go into an error state.

Autoscale had to handle such scenarios gracefully and the code was changed to
do so. And Mimic provided for a way to tests this consistently. 

However, like I said, we dont know what we dont know. We were not anticipating
any other such errors. But, there were more! And this was slow process for us
to uncover such errors as we tested against the real systems. And we continued
to add them to Mimic.

Now, Wont it be great if not every person writing an application that used
compute as a dependency had to go through this same cycle and have to find all
the possible error conditions in a system by experience and have to deal with
them at the pace that they occur.

What if we had a repository of all such known errors, that everyone
contributes to. So the next person using the plugin can use the existing ones,
and ensure there application behaves consistently irrespective of the errors,
and be able add any new ones to it.

Mimic is just that, a repository of all known responses including the error
responses.


# Lekha: call to action: why and how the openstack community should help

Mimic, can be the tool, where you do not have to stand up the entire dev stack
to understand how an OpenStack API behaves.

Mimic can be the tool which enables an OpenStack developer to get quick
feedback on the code he/she is writing and not have to go through the gate
multiple times to understand that, maybe I should have handled that one
error, that the upstream system decides to throw my way every now and then.









# deadcode
when the behavior of the upstream system changes, you only
need to make a change in one place - Mimic - instead of once in every test
that might be mocking different behavior of the upstream system.

This made it relatively straightforward to move from a model where that service
catalog was hard-coded based on a specific port-number offset for each service,
to one where a custom URL path was generated for each service.  Initially,
these were simply long opaque UUIDs per service, but Mimic now includes the
name of the endpoint in addition to a small random key, so that the URIs are
both unique and readable for debugging purposes.
