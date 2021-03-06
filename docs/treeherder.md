# Treeherder Integration

The core of mozilla-taskcluster is integrating with the pushlog and
treeherder. Treeherder integration is split into three parts:

 - [Pushlog monitoring](Pushlog)
 - [Actions](Actions)
 - [Task routes](Task Routing)

## Pushlog monitoring

The [pushlog_monitor](./src/bin/pushlog_monitor.js) is responsible for
converting pushlog entries into treeherder resultsets this process
closely mirrors the logic treeherder itself uses but is duplicated here
to ensure that the moment we notice a push we create a resultset and the
associated graph.

The Repositories collection contains the list of monitored repositories
(Note that this only is supported for repositories which live under the
hg.mozilla.org/* host with the pushlog extension.

## Actions

Treeherder UI presents the user with a number of options the two
important ones are retrigger and cancel. Whenever these actions are
requsted treeherder emits a pulse message in roughly the following form:

```
exchange: treeherder/v1/job-actions
routing key: <build_system_type>.<project>.<action>
schema: https://github.com/mozilla/treeherder-service/blob/e889dd1c951b925d621d6c4220a3566f834e0cfa/schemas/job-action-message.json
````

### The "retrigger" action

When a job requests a retrigger we duplicate it's graph dependant on
it's state:

##### Failed/Exception States

The node in the graph is duplicated as well as all _dependant_ nodes
this means if a build fails and you retrigger it the tests along with
the build are constructed into a new graph.

#### Other cases (pending/running/etc...)

We duplicte _only_ the single node in the graph (so retrigger completed
builds will not retrigger tests for that build)

## Task routes

Tasks have the ability to define new jobs/symbols in treeherder
this is enabled by adding the appriopriate route. The route is
constructed in the following form:

```
<prefix>.<project>.<revision_hash>
```

An example being:

```
tc-treeherder.b2g-inbound.e7e718507b7bcb1cebfb99a0ef6d170fa7ac4acd
```

(The above is the real resultset from https://treeherder.mozilla.org/#/jobs?repo=b2g-inbound&revision=8fed85cc71d5)

Generally the you don't need to know about the `revision_hash` it is
generated by the logic which creates graphs.

Currently there are two prefixes each prefix maps to one treeherder
environment:

| route prefix        | treeherder host |
|---------------------|------------------------|
| tc-treeherder       | treeherder.mozilla.org |
| tc-treeherder-stage | treeherder.allizom.org |

In addition to the route there are a number of properties which can be
attached to the task which will define how the symbol is created in the
treeherder UI.

```js
{
  // ... task stuff ...
  extra: {
    treeherder: {
      symbol: '...', // defines the symbol used in the treeherder UI
      groupSymbol: '...', // optionally defined this job as part of the group symbol
      groupName: '...', // optionally define group name

      // Define which build type this task is a part of note that this
      // structure maps directly to the treeherder API you _must_
      // specify one key but only one key (defaults to opt)
      collection: {
        opt: true // the default
        // debug: true (debug builds)
      },

      // Define the builder type
      machine: {
        // This _must_ be one of the values in https://github.com/mozilla/treeherder-service/blob/31acccb58082b3cbcfb8bc44c10d3c2346962701/treeherder/webapp/api/resultset.py#L19
        platform: '...'
      }
    }
  }
}
```

The states of taskcluster jobs map directly into the states defined by
treehreder:

| taskcluster state | treeherder state | symbol color |
| ----------------- | ---------------- | ------------ |
| defined           | pending          | faded grey   |
| pending           | pending          | faded grey   |
| running           | running          | grey         |
| success           | success+complete | green        |
| failed            | testfailed+complete | red       |
| exception         | exception+complete  | purple    |
| (on rerun)        | rerun+complete      | blue      |
