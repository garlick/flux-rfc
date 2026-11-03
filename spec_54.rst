.. github display
   GitHub is NOT the preferred viewer for this file. Please visit
   https://flux-framework.rtfd.io/projects/flux-rfc/en/latest/spec_54.html

54/Flux Job Dictionary
######################

This RFC defines the Flux job dictionary, a job-scoped key-value service
in the job shell, and the process manager use cases it must support.

-  Name: github.com/flux-framework/rfc/spec_54.rst

-  Editor: Jim Garlick <garlick@llnl.gov>

-  State: raw

Language
********

.. include:: common/language.rst

Related Standards
*****************

-  :doc:`spec_6`

-  :doc:`spec_13`

-  :doc:`spec_20`

-  :doc:`spec_22`

-  :doc:`spec_34`

-  `PMIx Standard, version 5.0
   <https://github.com/pmix/pmix-standard/releases/tag/v5.0>`_

Background
**********

Job shell plugins sometimes need to share data among the shells of a job.
Process managers exchange bootstrap data among the tasks they serve, and
the shell hands a nested Flux instance information about its parent.  The
builtin *jobdict* plugin provides named, job-scoped dictionaries with
collective exchange over all shells or a subset, through the shell plugin
API.

The primary users are the process manager interfaces: the builtin PMI-1
server and the PMIx server in flux-pmix.  Their consumers are chiefly
MPICH, Open MPI, and their derivatives, though the interfaces are not
limited to MPI.  This RFC treats them as use cases that the job
dictionary must support.  Other plugins MAY use the job dictionary to
exchange small amounts of data.

Goals
*****

-  Define the job dictionary data model and operations independently of
   any one client.

-  Define exchange semantics over all shells or a subset, and remote lookup
   from one shell, including concurrency, timeouts, and failure.

-  Define groups of task ranks and how they determine exchange
   participants.

-  Define the ``flux`` dictionary and the keys reserved in it.

-  Describe how the process manager interfaces use the job dictionary, and
   the requirements they place on it.

Non-goals: restating :doc:`spec_13` or the PMIx Standard; process manager
abstractions not built on the job dictionary, such as rank, size, and
abort; dynamic process management (spawn, name publishing, and
allocation), which is deferred to a future RFC; and process managers such
as cray-pals that bootstrap without the job dictionary.

Terminology
***********

Shell rank
  The rank of a job shell within the job, 0 to *size* - 1.

Task rank
  The rank of a task within the job, as assigned by the RFC 34 taskmap.

Job dictionary (dictionary)
  A named, job-scoped, ephemeral key-value store with one instance per
  shell.  Values are arbitrary JSON.

Local key
  A key put with the local flag.  It is visible only on the shell where it
  was put and is never exchanged.

Pending key
  A non-local key put since this shell's last exchange on the dictionary.

Exchange
  A collective operation among a set of participant shells that makes
  their pending keys visible to one another.

Participants
  The set of shell ranks taking part in an exchange.

Group
  A set of task ranks, optionally named.  The participants of an exchange
  over a group are the shells that host its members.

Reach
  A list of other dictionaries whose keys may be read, but not written,
  through a dictionary.

Pre-set key
  A key placed in a dictionary before any client runs, such as
  ``flux.taskmap``.

Requirements
************

The job dictionary SHALL meet the following requirements.  Each is drawn
from a use case and met by the section named.

.. list-table::
   :header-rows: 1

   * - #
     - Requirement
     - Needed by
     - Met by
     - Status
   * - R1
     - Several named dictionaries per job, created independently by different plugins
     - PMI-1, PMIx, other plugins
     - `Data Model`_
     - implemented
   * - R2
     - Local keys, visible only on the shell that put them
     - ``flux`` dictionary, ``PMI_process_mapping``
     - `Key Scope`_
     - implemented
   * - R3
     - Read access to another dictionary's keys, which clients cannot write or shadow
     - PMI-1, PMIx (``flux.*``)
     - `Reach`_
     - implemented
   * - R4
     - Usable by plugins outside flux-core through the shell plugin API alone
     - flux-pmix, other plugins
     - `Operations`_
     - implemented
   * - R5
     - Collective exchange over all shells
     - PMI-1 barrier, PMIx wildcard fence
     - `Exchange Semantics`_
     - implemented
   * - R6
     - Exchange over a subset that does not block other shells
     - PMI-1.2 group barrier, PMIx fence and group
     - `Exchange Semantics`_
     - implemented
   * - R7
     - Concurrent exchanges, distinguished by tag
     - PMI-1.2 outstanding barriers, PMIx
     - `Entering an Exchange`_
     - implemented
   * - R8
     - Failures that propagate, so no shell is left blocked
     - all
     - `Completion and Errors`_
     - implemented
   * - R9
     - Pre-set keys available to plugins from ``shell.init`` onward
     - nested Flux instances, PMI-1, PMIx
     - `The flux Dictionary`_
     - implemented
   * - R10
     - Values opaque to the job dictionary
     - PMIx (OpenPMIx data)
     - `Data Model`_
     - implemented *(verify encoding of binary data)*
   * - R11
     - Groups as unordered sets of task ranks, resolved to their hosting shells
     - PMI-1.2 (flux-core issue 7888), PMIx process arrays
     - `Groups`_
     - proposed
   * - R12
     - An exchange result holding only that exchange's contributions
     - PMIx
     - `Carry-forward`_
     - proposed
   * - R13
     - Timeouts on exchanges and lookups
     - PMI-1 shell option, PMIx ``PMIX_TIMEOUT``
     - `Timeouts`_
     - proposed
   * - R14
     - Asynchronous lookup of one key on one other shell, with on-demand supply
     - PMIx direct modex
     - `Remote Lookup`_
     - proposed
   * - R15
     - Job-wide unique IDs
     - PMIx group context IDs
     - none yet
     - open

Data Model
**********

A job MAY have several dictionaries.  Each is identified by a name that
SHALL be non-empty, SHALL NOT contain a period, and SHALL be unique within
the shell.  Every shell that uses a dictionary SHALL create it under the
same name before entering an exchange on it.  Clients SHALL NOT assume a
single dictionary per job.

Key Scope
=========

-  A key put without the local flag becomes pending and SHALL be carried
   in this shell's next exchange on the dictionary.

-  A key put with the local flag SHALL be visible only on this shell and
   SHALL NOT be exchanged.

-  A get SHALL return a key put on this shell, local or pending, without
   waiting for an exchange.

Reach
=====

A dictionary MAY name other dictionaries in its reach at create time.  A
key of the form ``OTHER.SUBKEY``, where ``OTHER`` is in reach, SHALL be
served by looking up ``SUBKEY`` in dictionary ``OTHER``.

Such keys SHALL be read-only through the reaching dictionary: a put to
them SHALL fail, and a value put under the same name in the reaching
dictionary SHALL NOT shadow the reached one.  This lets a client pass keys
owned by another plugin through to its own consumers without knowing their
names, and prevents those consumers from overriding them.

Operations
**********

The job shell SHALL provide these operations through the plugin stack.
Each takes a JSON object of input arguments and, on failure, sets an
*errmsg* string in the output arguments.

.. list-table::
   :header-rows: 1

   * - Topic
     - Inputs
     - Outputs
     - Notes
   * - ``jobdict.create``
     - name:s, k?:i, reach?:[s]
     - name:s
     - *k* is the exchange tree fanout, default 2
   * - ``jobdict.put``
     - name:s, key:s, value:o, local?:b
     - none
     - *value* is any JSON value
   * - ``jobdict.get``
     - name:s, key:s
     - value?:o
     - a missing key succeeds with no value
   * - ``jobdict.exchange``
     - name:s, tag:s, participants?:s, exit_topic:s
     - name:s
     - *participants* is an RFC 22 idset of shell ranks; completion
       arrives on *exit_topic*

Because a plugin stack call succeeds when no plugin handles the topic,
``jobdict.create`` and ``jobdict.exchange`` SHALL return *name* on success.
A caller SHALL treat its absence as an indication that the job dictionary
is not available.

This RFC proposes four additions, each described below: an optional
*timeout* on ``jobdict.exchange`` (see `Timeouts`_), a *carry* flag on
``jobdict.create`` (see `Carry-forward`_), a *provider_topic* on
``jobdict.create``, and a new ``jobdict.lookup`` operation (both in
`Remote Lookup`_).

Exchange Semantics
******************

An exchange makes the pending keys of its participants visible to one
another.  It is the only synchronization primitive the job dictionary
provides.

Entering an Exchange
====================

-  If *participants* is omitted, all shells participate.

-  The calling shell SHALL be a participant, and every participant SHALL
   be a valid shell rank.

-  An exchange is identified by its dictionary and tag.  Every participant
   SHALL enter it with the same dictionary name, tag, and participant set.
   Clients SHALL derive tags that all participants compute identically.

-  Exchanges with different tags on the same dictionary SHALL proceed
   independently and MAY be in flight concurrently.

-  An exchange SHALL NOT be entered before ``shell.post-init``, since it
   sends RPCs to other shells.

-  The pending keys carried by an exchange SHALL be those pending when
   this shell enters it.  A key put after entry SHALL remain pending for a
   later exchange.

Visibility
==========

-  When an exchange among all shells completes, its keys SHALL be visible
   on every shell.

-  When an exchange among a subset completes, its keys SHALL be visible on
   each shell in the subset.

-  Keys exchanged within a subset SHALL be carried forward and included in
   the next exchange among all shells, so that they eventually become
   visible everywhere.

Keys exchanged in one subset are not guaranteed to be visible in a
different, overlapping subset until a full exchange has occurred.  These
rules match the PMI-1.2 group barrier semantics in :doc:`spec_13`.

Carry-forward
=============

This section is a proposal.

Carry-forward suits PMI-1, where a key put once is expected to become
visible everywhere eventually.  It does not suit PMIx, where each fence's
result must hold only the data contributed to that fence; carried-forward
data would hand OpenPMIx stale blobs in a later fence.

-  ``jobdict.create`` SHALL accept *carry*:b, default true, which gives the
   behavior above.

-  When *carry* is false, keys exchanged within a subset SHALL NOT be
   carried into later exchanges, and an exchange's result SHALL hold only
   keys contributed to that exchange.

Whether today's exchange result includes carried-forward keys should be
checked against the code.

Completion and Errors
=====================

On completion, the shell SHALL call *exit_topic* through the plugin stack
with (name:s, tag:s, result:o) on success, where *result* holds the keys
exchanged, or (name:s, tag:s, errmsg:s) on failure.

-  If an exchange fails on a shell, that shell SHALL respond with an error
   to any child requests it has queued, so the failure propagates instead
   of leaving other shells blocked.

-  If entering fails, for example because more child requests arrived
   than the tree allows, the shell SHALL fail its queued child requests
   and discard the exchange.

-  A failed exchange among a subset MUST NOT leave non-participating
   shells blocked.

Timeouts
========

This section is a proposal.

-  ``jobdict.exchange`` and ``jobdict.lookup`` SHALL accept an optional
   *timeout*:f in seconds.  Without one they wait indefinitely, as today.

-  Participants in an exchange SHOULD use the same timeout.

-  When the timeout expires on a shell before the exchange completes, that
   shell SHALL fail the exchange locally with an *errmsg* naming the
   timeout, and SHALL respond with an error to any child requests it has
   queued.  A response arriving later for that exchange SHALL be
   discarded.

-  A timeout fails the exchange, not the job.  The client decides what to
   do, such as returning an error to its own caller or raising a job
   exception.

Exchange Protocol
=================

This section is informative.  It MAY become normative so that
independent implementations interoperate.

flux-core places each participant in a k-ary tree over the sorted
participant set, with *k* set at dictionary create.  A single shell
service handles exchange requests for all dictionaries and routes each by
the dictionary name in its payload.  A shell gathers its children's
pending keys, merges its own, and sends the result to its parent in a
request carrying the dictionary name, tag, and a ``{data}`` envelope.  The
merged result returns down the tree in the responses.  Since a shell
cannot know its child count until it enters, it accepts at most *k*
queued child requests before entry.  *(verify service name and payload
fields)*

Remote Lookup
*************

This section is a proposal.

A remote lookup fetches one key from one other shell, without a
collective.  It exists for PMIx direct modex, where a client reads data
for a process it never fenced with, but any plugin MAY use it.

.. list-table::
   :header-rows: 1

   * - Topic
     - Inputs
     - Outputs
     - Notes
   * - ``jobdict.lookup``
     - name:s, key:s, shell:i, timeout?:f, exit_topic:s
     - name:s
     - completion arrives on *exit_topic*

-  The request SHALL be sent to the shell with rank *shell*.  Once groups
   are defined, a task rank MAY be given instead and resolved to the shell
   that hosts it.

-  The target SHALL answer from keys put on it, pending or already
   exchanged.  It SHALL NOT return local keys.

-  If the key is absent, the target SHALL hold the request until the key
   is put or the timeout expires, rather than fail at once.  In PMIx, the
   target process may not have committed its data yet.

-  ``jobdict.create`` SHALL accept an optional *provider_topic*.  When a
   lookup finds a key absent on a dictionary that has one, the target
   SHALL call the provider topic through the plugin stack with (name:s,
   key:s).  The provider MAY then put the key asynchronously, which
   completes every lookup waiting on it.  flux-pmix uses this to ask
   OpenPMIx for the requested process's data.

-  On completion, the caller's *exit_topic* SHALL be called with (name:s,
   key:s, value:o) on success or (name:s, key:s, errmsg:s) on failure.

-  Lookups SHALL be asynchronous and MAY be in flight concurrently with one
   another and with exchanges.

-  A looked-up value SHALL be returned to the caller only.  It SHALL NOT be
   stored in the caller's dictionary or change visibility.

Groups
******

This section is a proposal.  Today, exchange participants are given as
shell ranks, and each client translates its own notion of a group into
them.

The job dictionary SHOULD accept a group in place of explicit
participants, so that clients share one definition:

-  A group SHALL be a set of task ranks.  Order and duplicates SHALL NOT
   matter.

-  The participants of an exchange over a group SHALL be the shells that
   host its members, as given by the RFC 34 taskmap.

-  A group MAY be anonymous, given as an RFC 22 idset of task ranks, or
   named.

-  Named groups SHALL be published as local pre-set keys in the ``flux``
   dictionary, so that every client reports the same names and
   membership.

-  Group names defined by Flux SHALL use a single reserved prefix (to be
   chosen) and SHALL NOT use the ``mpi://`` prefix, which belongs to MPI,
   or the ``pmix`` prefix, which the PMIx Standard reserves.

Named groups give MPI Sessions process sets beyond ``mpi://WORLD`` and
``mpi://SELF``.  Candidates are one per node and one per application in a
multi-application job.

The flux Dictionary
*******************

The builtin fluxkeys plugin SHALL create a dictionary named ``flux`` at
``shell.init`` and populate it with the following local keys, computed
identically on every shell:

.. list-table::
   :header-rows: 1

   * - Key
     - Value
   * - ``taskmap``
     - RFC 34 taskmap of the job
   * - ``instance-level``
     - nesting depth of the job's Flux instance
   * - ``tbon-interface-hint``
     - network interface hint for a child broker overlay
   * - ``constrained-resources``
     - whether the job's resources are constrained

A dictionary that reaches ``flux`` exposes these as ``flux.taskmap``,
``flux.instance-level``, and so on.  Since builtin plugins initialize
first, the keys SHALL be available to other plugins from their own
``shell.init`` callback onward.  Named groups, once defined, SHALL also be
published here.

Use Cases
*********

PMI-1.x
=======

The builtin PMI-1 server implements the :doc:`spec_13` wire protocol.  It
keeps its KVS in a job dictionary of its own, created with ``flux`` in
its reach.  Requests not listed below, such as rank, size, and appnum, are
answered from shell information and are outside this RFC.

.. list-table::
   :header-rows: 1

   * - PMI-1 request
     - Job dictionary operation
     - Notes
   * - ``put``
     - ``jobdict.put``, not local
     - one dictionary per job regardless of kvsname *(verify)*
   * - ``get``
     - ``jobdict.get``
     - ``flux.KEY`` resolves through reach
   * - ``barrier_in``, no group
     - exchange over all shells
     - legacy barrier; ``barrier_out`` carries no tag
   * - ``barrier_in group=WORLD:N``
     - exchange over all shells
     - tag is the group string
   * - ``barrier_in group=NODE:N``
     - exchange over this shell only
     - still an exchange, so pending keys are merged
   * - ``barrier_in group=r1,r2,...:N``
     - exchange over the group r1, r2, ...
     - ``barrier_out`` echoes ``tag=N``

The PMI-1 server depends on requirements R1 to R3, R5 to R9, R11, and
R13.  A proposed shell option, for example ``pmi-simple.timeout``, would
bound how long a barrier waits; on expiry the server returns a nonzero
``rc`` in ``barrier_out``.

The server additionally reports a value longer than the PMI-1 maximum as
not found rather than truncating it, hides ``flux.taskmap`` when
``pmi-simple.nomap`` is set, and pre-sets ``PMI_process_mapping`` as
defined in :doc:`spec_13` *(verify location)*.

PMIx
====

The flux-pmix plugin implements a PMIx server with the OpenPMIx server
library, on a job dictionary of its own created with ``flux`` in its
reach.  OpenPMIx keeps client keys in its own store and has no server
upcall for individual puts and gets, so flux-pmix maps PMIx collectives
onto exchanges that carry the opaque data OpenPMIx supplies.  Key naming,
put scope, and what a client may read after a fence are as defined by the
PMIx Standard and enforced by OpenPMIx.

.. list-table::
   :header-rows: 1

   * - PMIx server upcall
     - Job dictionary operation
     - Notes
   * - ``fence_nb``
     - exchange over the group of listed processes
     - the OpenPMIx data is this shell's pending value; tag derived as below
   * - ``fence_nb`` over ``PMIX_RANK_WILDCARD``
     - exchange over all shells
     - 
   * - ``fence_nb`` without ``PMIX_COLLECT_DATA``
     - exchange carrying no data
     - still synchronizes the participants *(verify)*
   * - ``fence_nb`` with ``PMIX_TIMEOUT``
     - exchange with *timeout*
     - proposed
   * - ``direct_modex``
     - ``jobdict.lookup`` on the shell hosting the process
     - the target's *provider_topic* asks OpenPMIx for the data; proposed
   * - ``group``, construct
     - exchange over the member group
     - tag is the group name; context ID as below
   * - ``group``, destruct
     - exchange over the member group, no data
     - 
   * - job-level information at namespace registration
     - ``jobdict.get`` of ``flux.*`` keys through reach
     - exposed under the same names *(verify)*; resolves flux-pmix issue 135

The PMIx server depends on requirements R1 to R15.  In addition,
flux-pmix SHALL follow these rules:

-  Every process named in a fence or group SHALL be in the job's own
   namespace.  A request naming another namespace SHALL fail, since
   connect and spawn are deferred.

-  ``fence_nb`` carries no tag.  flux-pmix SHALL derive one from the
   canonical RFC 22 idset of the participating task ranks plus a per-set
   sequence number.  This relies on the PMIx requirement that participants
   call fences over the same set in the same order.

-  flux-pmix SHALL create its dictionary with *carry* false, so a fence's
   result holds only that fence's data.

-  flux-pmix SHALL use a dictionary separate from the PMI-1 server's,
   since the OpenPMIx key space cannot be shared with it *(verify
   dictionary name)*.

-  Group context IDs requested with ``PMIX_GROUP_ASSIGN_CONTEXT_ID`` SHALL
   come from a job-wide allocator (R15).  The exchange tree root cannot
   allocate them, because which shell is the root depends on the
   participants.

-  ``PMIx_Group_invite()`` and join, and group changes on member failure,
   are not collectives over a fixed membership and are out of scope.

Other Shell Plugins
===================

A plugin MAY create its own dictionary to share small amounts of data
among shells, for example to gather per-node information or publish a
value computed on one shell.  It SHALL NOT enter an exchange before
``shell.post-init``, and SHOULD put large data elsewhere, such as the job's
KVS namespace.

Process Managers That Do Not Use the Job Dictionary
===================================================

The cray-pals plugin in flux-coral2 prepares the apinfo file and
environment that Cray PMI uses to bootstrap itself, and Cray PMI does its
own data exchange.  It MAY be enabled alongside the servers above, for
example with ``pmi=cray-pals,simple``.

Open Issues
***********

-  **Groups API.**  Whether ``jobdict.exchange`` takes a group directly,
   and how named groups are written in the ``flux`` dictionary.

-  **Group name prefix and contents.**  Pick the prefix and decide which
   named groups to define.

-  **Tag agreement in RFC 13.**  RFC 13 says a tag SHOULD be unique
   within a client, but an exchange requires all members to use the same
   tag.  Confirm MPICH guarantees this and say so in RFC 13.

-  **NODE wording in RFC 13.**  The barrier operation section describes
   ``NODE`` as "this rank"; the API table describes the processes on the
   caller's node.

-  **Overlapping subsets.**  Decide whether to promise anything about
   visibility in an overlapping subset before a full exchange.

-  **Exchange protocol.**  Decide whether to make it normative.

-  **PMIx group context IDs.**  Where the job-wide allocator lives, for
   example the leader shell or the plugin that owns the ``flux``
   dictionary.

-  **Lookup provider failures.**  Whether a provider error fails waiting
   lookups at once, or only at their timeout.

-  **Timeout options.**  Names and defaults for the PMI-1 server option and
   any jobdict-wide default.

To Verify Against the Code
==========================

-  Name of the PMI-1 dictionary and the kvsname returned by
   ``get_my_kvsname``.

-  Tag used for the legacy (no group) barrier exchange.

-  Where ``PMI_process_mapping`` is pre-set.

-  Exchange service name and request payload fields.

-  flux-pmix dictionary name, fence tag derivation, behavior without
   ``PMIX_COLLECT_DATA``, and names of ``flux.*`` keys exposed to clients.

References
**********

-  `flux-core PR 7856: shell: factor out PMI kvs for use by other process
   managers <https://github.com/flux-framework/flux-core/pull/7856>`_

-  `flux-core PR 7862: shell: add support for PMI-1.2 and group barriers
   <https://github.com/flux-framework/flux-core/pull/7862>`_

-  `flux-core issue 7888: PMI-1.2 group barrier does not support an
   unordered list of ranks
   <https://github.com/flux-framework/flux-core/issues/7888>`_

-  `flux-shell-plugins(7), Job Dictionaries
   <https://flux-framework.readthedocs.io/projects/flux-core/en/latest/man7/flux-shell-plugins.html>`_

-  `flux-shell-cray-pals(1), flux-coral2
   <https://flux-framework.readthedocs.io/projects/flux-coral2/en/latest/man1/flux-shell-cray-pals.html>`_

-  `flux-pmix issue 135: pre-populate KVS keys for bootstrapping flux
   brokers <https://github.com/flux-framework/flux-pmix/issues/135>`_
