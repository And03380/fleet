# Using Fleet FAQ

## How can I switch to Fleet from Kolide Fleet?

To migrate to Fleet from Kolide Fleet, please follow the steps outlined in the [Upgrading Fleet section](https://fleetdm.com/docs/deploying/upgrading-fleet) of the documentation.

## Has anyone stress tested Fleet? How many hosts can the Fleet server handle?

Fleet has been stress tested to 150,000 online hosts and 400,000 total enrolled hosts. Production deployments exist with over 100,000 hosts and numerous production deployments manage tens of thousands of hosts.

It’s standard deployment practice to have multiple Fleet servers behind a load balancer. However, typically the MySQL database is the performance bottleneck and a single Fleet server can handle tens of thousands of hosts.

## Can I target my hosts using their enroll secrets?

No, currently, there’s no way to retrieve the name of the enroll secret with a query. This means that there's no way to create a label using your hosts' enroll secrets and then use this label as a target for live queries or scheduled queries.

Typically folks will use some other unique identifier to create labels that distinguish each type of device. As a workaround, [Fleet's manual labels](https://fleetdm.com/docs/using-fleet/fleetctl-cli#host-labels) provide a way to create groups of hosts without a query. These manual labels can then be used as targets for queries.

There is, however, a way to accomplish this even though the answer to the question remains "no": Teams. As of Fleet v4.0.0, you can group hosts in Teams either by enrolling them with a team specific secret, or by transferring hosts to a team. One the hosts you want to target are part of a team, you can create a query and target the team in question.

## How often do labels refresh? Is the refresh frequency configurable?

The update frequency for labels is configurable with the [—osquery_label_update_interval](https://fleetdm.com/docs/deploying/configuration#osquery-label-update-interval) flag (default 1 hour).

## How do I revoke the authorization tokens for a user?

Authorization tokens are revoked when the “require password reset” action is selected for that user. User-initiated password resets do not expire the existing tokens.

## How do I monitor the performance of my queries?

Fleet can live query the `osquery_schedule` table. Performing this live query allows you to get the performance data for your scheduled queries. Also consider scheduling a query to the `osquery_schedule` table to get these logs into your logging pipeline.

## How do I monitor a Fleet server?

Fleet provides standard interfaces for monitoring and alerting. See the [Monitoring Fleet](https://fleetdm.com/docs/using-fleet/monitoring-fleet) documentation for details.

## Why is the “Add User” button disabled?

The “Add User” button is disabled if SMTP (email) has not been configured for the Fleet server. Currently, there is no way to add new users without email capabilities.

One way to hack around this is to use a simulated mailserver like [Mailhog](https://github.com/mailhog/MailHog). You can retrieve the email that was “sent” in the Mailhog UI, and provide users with the invite URL manually.

## Can I disable password-based authentication in the Fleet UI?

Some folks like to enforce users with SAML SSO enabled to login only via the SSO and not via password.

There is no option in the Fleet UI for disabling password-based authentication.
However, users that have SSO enabled in Fleet will not be able to log in via password-based authentication.

If a user has SSO enabled, the Login page in the Fleet UI displays the “Email” and “Password” fields but on attempted password-based login, this user will receive an “Authentication failed” message.

## Where are my query results?

### Live queries

Live query results (executed in the web UI or `fleetctl query`) are pushed directly to the UI where the query is running. The results never go to a file unless you as the user manually save them.

### Scheduled queries

Scheduled query results from enrolled hosts can be logged by Fleet.
For results to go to Fleet, the osquery `--logger_plugin` flag must be set to `tls`.

### What are my options for storing the osquery logs?

Folks typically use Fleet to ship logs to data aggregation systems like Splunk, the ELK stack, and Graylog.

Fleet supports multiple logging destinations for scheduled query results and status logs. The `--osquery_result_log_plugin` and `--osquery_status_log_plugin` can be set to:
`filesystem`, `firehose`, `kinesis`, `lambda`, `pubsub`, `kafkarest`, and `stdout`.
See:
  - https://fleetdm.com/docs/deploying/configuration#osquery-result-log-plugin.
  - https://fleetdm.com/docs/deploying/configuration#osquery-status-log-plugin.

### Troubleshooting

Expecting results, but not seeing anything in the logs?

- Try scheduling a query that always returns results (eg. `SELECT * FROM time`).
- Check whether the query is scheduled in differential mode. If so, new results will only be logged when the result set changes.
- Ensure that the query is scheduled to run on the intended platforms, and that the tables queried are supported by those platforms.
- Use live query to `SELECT * FROM osquery_schedule` to check whether the query has been scheduled on the host.
- Look at the status logs provided by osquery. In a standard configuration these are available on the filesystem of the Fleet server at the path configurable by [`--filesystem_status_log_file`](https://fleetdm.com/docs/deploying/configuration#filesystem-status-log-file). This defaults to `/tmp/osquery_status`. The host will output a status log each time it executes the query.

## Why does the same query come back faster sometimes?

Don't worry, this behavior is expected; it's part of how osquery works.

Fleet and osquery work together by communicating with heartbeats. Depending on how close the next heartbeat is, Fleet might return results a few seconds faster or slower.
>By the way, to get around a phenomena called the "thundering herd problem", these heartbeats aren't exactly the same number of seconds apart each time. osquery implements a "splay", a few ± milliseconds that are added to or subtracted from the heartbeat interval to prevent these thundering herds. This helps prevent situations where many thousands of devices might unnecessarily attempt to communicate with the Fleet server at exactly the same time. (If you've ever used Socket.io, a similar phenomena can occur with that tool's automatic WebSocket reconnects.)

## Why don't my query results appear sorted based upon the ORDER BY clause I specified in my SQL query? 

When a query executes in Fleet, the query is sent to all hosts at the same time, but results are returned from hosts at different times. In Fleet, results are shown as soon as Fleet receives a response from a host. Fleet does not sort the overall results across all hosts (the sort UI toggle is used for this). Instead, Fleet prioritizes speed when displaying the results.  This means that if you use an `ORDER BY` clause selection criteria in a query, the results may not initially appear with your desired order, however, the sort UI toggle allows you to sort by ascending or descending order for any of the displayed columns.

## What happens if I have a query on a team policy and I also have it scheduled to run separately?

Both queries will run as scheduled on applicable hosts. If there are any hosts that both the scheduled run and the policy apply to, they will be queried twice.

## Why aren’t my live queries being logged?

Live query results are never logged to the filesystem of the Fleet server. See [Where are my query results?](#where-are-my-query-results).

## Why does my query work locally with osquery but not in Fleet?

If you're seeing query results using `osqueryi` but not through Fleet, the most likely culprit is a permissions issue. Check out the [osquery docs](https://osquery.readthedocs.io/en/stable/deployment/process-auditing/#full-disk-access) for more details and instructions for setting up Full Disk Access.

## Can I use the Fleet API to fetch results from a scheduled query?

You cannot. Scheduled query results are logged to whatever logging plugin you have configured and are not stored in the Fleet DB.

However, the Fleet API exposes a significant amount of host information via the [`api/v1/fleet/hosts`](https://fleetdm.com/docs/using-fleet/rest-api#list-hosts) and the [`api/v1/fleet/hosts/{id}`](https://fleetdm.com/docs/using-fleet/rest-api#get-host) API endpoints. The `api/v1/fleet/hosts` [can even be configured to return additional host information](https://github.com/fleetdm/fleet/blob/9fb9da31f5462fa7dda4819a114bbdbc0252c347/docs/1-Using-Fleet/2-fleetctl-CLI.md#fleet-configuration-options).

For example, let's say you want to retrieve a host's OS version, installed software, and kernel version:

Each host’s OS version is available using the `api/v1/fleet/hosts` API endpoint. [Check out the API documentation for this endpoint](https://fleetdm.com/docs/using-fleet/rest-api#list-hosts).

The ability to view each host’s installed software was released behind a feature flag in Fleet 3.11.0 and called Software inventory. [Check out the feature flag documentation for instructions on turning on Software inventory in Fleet](https://fleetdm.com/docs/deploying/configuration#feature-flags).

Once the Software inventory feature is turned on, a list of a specific host’s installed software is available using the `api/v1/fleet/hosts/{id}` endpoint. [Check out the documentation for this endpoint](https://fleetdm.com/docs/using-fleet/rest-api#get-host).

It’s possible in Fleet to retrieve each host’s kernel version, using the Fleet API, through `additional_queries`. The Fleet configuration options YAML file includes an `additional_queries` property that allows you to append custom query results to the host details returned by the `api/v1/fleet/hosts` endpoint. [Check out an example configuration file with the additional_queries field](https://fleetdm.com/docs/using-fleet/fleetctl-cli#fleet-configuration-options).

## Why is my host not updating a policy's response?

The following are reasons why a host may not be updating a policy's response:

* The policy's query includes tables that are not compatible with this host's platform. For example, if your policy's query contains the [`apps` table](https://osquery.io/schema/5.0.1/#apps), which is only compatible on hosts running macOS, this policy will not update its response if this host is running Windows or Linux.

* The policy's query includes invalid SQL syntax. If your policy's query includes invalid syntax, this policy will not update its response. You can check the syntax of your query by heading to the **Queries** page, selecting your query, and then selecting "Save."

## What should I do if my computer is showing up as an offline host?

If your device is showing up as an offline host in the Fleet instance, and you're sure that the computer has osquery running, we recommend trying the following:

* Try un-enrolling and re-enrolling the host. You can do this by uninstalling osquery on the host and then enrolling your device again using one of the [recommended methods](https://fleetdm.com/docs/using-fleet/adding-hosts).

## How does Fleet deal with IP duplication?

Fleet relies on UUIDs so any overlap with host IP addresses should not cause a problem. The only time this might be an issue is if you are running a query that involves a specific IP address that exists in multiple locations as it might return multiple results - [Fleet's teams feature](https://fleetdm.com/docs/using-fleet/teams) can be used to restrict queries to specific hosts.

## Can fleetd run alongside osquery?

Yes, fleetd can be run alongside an existing, separately-installed osqueryd. If you have an existing osqueryd installed on a given host, you don't have to remove it prior to installing fleetd.  The osquery instance provided by fleetd uses its own database directory that doesn't interfere with other osquery isntances installed on the host.

## Can I control how fleetd handles updates?

Yes, auto-updates can be disabled entirely by passing `--disable-updates` as a flag when running `fleetctl package` to generate your installer (easy) or by deploying a modified systemd file to your hosts (more complicated). We'd recommend the flag:

```
fleetctl package --fleetctl package --type=deb --fleet-url=https://localhost:8080 --enroll-secret=superRandomSecret --disable-updates
```

You can also indicate the [channels you would like Orbit to watch for updates](https://github.com/fleetdm/fleet/tree/main/orbit#update-channels) using the `--orbit-channel`, `--desktop-channel` , and `--osqueryd-channel` flags:

```
fleetctl package --fleetctl package --type=deb --fleet-url=https://localhost:8080 --enroll-secret=superRandomSecret --orbit-channel=edge --desktop-channel=stable --osquery-channel=4
```

You can specify a major (4), minor (4.0) or patch (4.6.0) version as well as the `stable`  or `edge` channels.

## When will the newest version of osquery be available to Orbit?

When a new osquery version is released, it is pushed to the `edge` channel for beta testing. As soon as that version is deemed stable by the osquery project, it is moved to the `stable` channel. Some versions may take a little longer than others to be tested and moved from `edge` to `stable`, especially when there are major changes.

## Where does Orbit get update information?

Orbit checks for update metadata and downloads binaries at `tuf.fleetctl.com`.

## Can I bundle osquery extensions into Orbit?

This isn't supported yet, but we're working on it!

## What happens to osquery logs if my Fleet server or my logging destination is offline?

If Fleet can't send logs to the destination, it will return an error to osquery. This causes osquery to retry sending the logs. The logs will then be stored in osquery's internal buffer until they are sent successfully, or they get expired if the `buffered_log_max`(defaults to 1,000,000 logs) is exceeded. Check out the [Remote logging buffering section](https://osquery.readthedocs.io/en/latest/deployment/remote/#remote-logging-buffering) on the osquery docs for more on this behavior.

## How does Fleet work with osquery extensions?

Any extension table available in a host enrolled to Fleet can be queried by Fleet. Note that the "compatible with" message may show an error because it won't know your extension table, but the query will still work, and Fleet will gracefully ignore errors from any incompatible hosts.

## Why do I see "Unknown Certificate Error" when adding hosts to my dev server?

If you are using a self-signed certificate on `localhost`, add the  `--insecure` flag when building your installation packages:

```
fleetctl package --fleetctl package --type=deb --fleet-url=https://localhost:8080 --enroll-secret=superRandomSecret --insecure
```

## Can I hide known vulnerabilities that I feel are insignificant?

This isn't currently supported, but we're working on it! You can track that issue [here](https://github.com/fleetdm/fleet/issues/3152).

## Can I create reports based on historical data in Fleet?

Currently, Fleet only stores the current state of your hosts (when they last communicated with Fleet). The best way at the moment to maintain historical data would be to use the [REST API](https://fleetdm.com/docs/using-fleet/rest-api) or the [`fleetctl` CLI](https://fleetdm.com/docs/using-fleet/fleetctl-cli) to retrieve it manually. Then save the data you need to your schedule.

## When do I need fleetctl vs. the REST API vs. the Fleet UI?

[fleetctl](https://fleetdm.com/docs/using-fleet/fleetctl-cli) is great for users that like to do things in a terminal (like iTerm on a Mac). Lots of tech folks are real power users of the terminal. It is also helpful for automating things like deployments.

The [REST API](https://fleetdm.com/docs/using-fleet/rest-api) is somewhat similar to fleetctl, but it tends to be used more by other computer programs rather than human users (although humans can use it too). For example, our [Fleet UI](https://fleetdm.com/docs/using-fleet/rest-api) talks to the server via the REST API. Folks can also use the REST API if they want to build their own programs that talk to the Fleet server.

The [Fleet UI](https://fleetdm.com/docs/using-fleet/fleet-ui) is built for human users to make interfacing with the Fleet server user-friendly and visually appealing. It also makes things simpler and more accessible to a broader range of users.

## Why can't I run queries with `fleetctl` using a new API-only user?

In versions prior to Fleet 4.13, a password reset is needed before a new API-only user can perform queries. You can find detailed instructions for setting that up [here](https://github.com/fleetdm/fleet/blob/a1eba3d5b945cb3339004dd1181526c137dc901c/docs/Using-Fleet/fleetctl-CLI.md#reset-the-password).

## Can I audit actions taken in Fleet?

The [REST API `activities` endpoint](https://fleetdm.com/docs/using-fleet/rest-api#activities) provides a full breakdown of actions taken on queries, policies, and teams (Available in Fleet Premium) through the UI, the REST API, or `fleetctl`.

## How often is the software inventory updated?

By default, Fleet will query hosts for software inventory hourly. If you'd like to set a different interval, you can update the [periodicity](https://fleetdm.com/docs/deploying/configuration#periodicity) in your vulnerabilities configuration.

## Can I group results from multiple hosts?

There are a few ways you can go about getting counts of hosts that meet specific criteria using the REST API. You can use [`GET /api/v1/fleet/hosts`](https://fleetdm.com/docs/using-fleet/rest-api#list-hosts) or the [`fleetctl` CLI](https://fleetdm.com/docs/using-fleet/fleetctl-cli#available-commands) to gather a list of all hosts and then work with that data however you'd like. For example, you could retrieve all hosts using `fleetctl get hosts` and then use `jq` to pull out the data you need. The following example would give you a count of hosts by their OS version:

```
$ fleetctl get hosts --json | jq '.spec .os_version' | sort | uniq -c

   1 "CentOS Stream 8.0.0"
   2 "Ubuntu 20.4.0"
   1 "macOS 11.5.2"
   1 "macOS 11.6.3"
   1 "macOS 12.1.0"
   3 "macOS 12.2.1"
   3 "macOS 12.3.0"
   6 "macOS 12.3.1"
```

## How do I downgrade from Fleet Premium to Fleet Free?

> If you'd like to renew your Fleet Premium license key, please contact us [here](https://fleetdm.com/company/contact).

**Back up your users and update all team-level users to global users**

1. Run the `fleetctl get user_roles > user_roles.yml` command. Save the `user_roles.yml` file so that, if you choose to upgrade later, you can restore user roles.
2. Head to the **Settings > Users** page in the Fleet UI.
3. For each user that has any team listed under the **Teams** column, select **Actions > Edit**, then select **Global user**, and then select **Save**. If a user shouldn't have global access, delete this user.

**Move all team-level scheduled queries to the global level**

1. Head to the **Schedule** page in the Fleet UI.
2. For each scheduled query that belongs to a team, copy the name in the **Query** column, select **All teams** in the top dropdown, select **Schedule a query**, past the name in the **Select query** field, choose the frequency, and select **Schedule**.
3. Delete each scheduled query that belongs to a team because they will no longer run on any hosts following the downgrade process.

**Move all team level policies to the global level**

1. Head to the **Policies** page in the Fleet UI.
2. For each policy that belongs to a team, copy the **Name**, **Description**, **Resolve**, and **Query**. Then, select **All teams** in the top dropdown, select **Add a policy**, select **create your own policy**, paste each item in the appropriate field, and select **Save**.
3. Delete each policy that belongs to a team because they will no longer run on any hosts following the downgrade process.

**Back up your teams**

1. Run the `fleetctl get teams > teams.yml` command. Save the `teams.yml` file so that, if you choose to upgrade later, you can restore teams.
2. Head to the **Settings > Teams** page in the Fleet UI.
3. Delete all teams. This will move all hosts to the global level.

**Remove your Fleet Premium license key**

1. Remove your license key from your Fleet configuration. Documentation on where the license key is located in your configuration is [here](https://fleetdm.com/docs/deploying/configuration#license).
2. Restart your Fleet server.

## If I use a software orchestration tool (Ansible, Chef, Puppet, etc.) to manage agent options, do I have to apply the same options in the Fleet UI?

No. The agent options set using your software orchestration tool will override the default agent options that appear in the **Settings > Organization settings > Agent options** page. On this page, if you hit the **Save** button, the options that appear in the Fleet UI will override the agent options set using your software orchestration.

## How can I uninstall the osquery agent?
To uninstall the osquery agent, follow the below instructions for your operating system.

### MacOS
Run the Orbit [cleanup script](https://github.com/fleetdm/fleet/blob/main/orbit/tools/cleanup/cleanup_macos.sh)

### Windows
Use the "Add or remove programs" dialog to remove Orbit.

### Ubuntu
Run `sudo apt remove fleet-osquery -y`

### CentOS
Run `sudo rpm -e fleet-osquery-X.Y.Z.x86_64`

## How does Fleet determines online and offline status?

### Online hosts

**Online** hosts will respond to a live query.

A host is online if it has connected successfully in a window of time set by `distributed_interval` (or `config_tls_refresh`, whichever is smaller).
A buffer of 60 seconds is added to the calculation to avoid unnecessary flapping between online/offline status (in case hosts take a bit longer than expected to connect to Fleet).
The values for `distributed_interval` and `config_tls_refresh` can be found in the **Settings > Organization settings > Agent options** page for global hosts
and in the **Settings > Teams > TEAM NAME > Agent options** page for hosts that belong to a team.

For example:

`distributed_interval=10, config_tls_refresh=30`
A host is considered online if it has connected to Fleet in the last 70 (10+60) seconds.

`distributed_interval=30, config_tls_refresh=20`
A host is considered online if it has connected to Fleet in the last 80 (20+60) seconds.

### Offline hosts

**Offline** hosts won't respond to a live query. These hosts may be shut down, asleep, or not connected to the internet.
A host could also be offline if there is a connection issue between the osquery agent running in the host and Fleet (see [What should I do if my computer is showing up as an offline host?](#what-should-i-do-if-my-computer-is-showing-up-as-an-offline-host)).

## Why aren't "additional queries" being applied to hosts enrolled in a team?

Changes were introduced in Fleet v4.20.0 that caused the `features.additional_queries` set in at the global level to no longer apply to hosts assigned to a team. If you would like those queries to be applied to hosts assigned to a team, you will need to be include these queries under `features.additional_queries` in each team's [configuration](https://fleetdm.com/docs/using-fleet/configuration-files#teams).

## Why am I seeing an error when using the `after` key in `api/v1/fleet/hosts`?

There is a [bug](https://github.com/fleetdm/fleet/issues/8443) in MySQL validation in some versions of Fleet when using the `created_at` and `updated_at` columns as `order_key` along with an `after` filter. Adding `h.` to the column in `order_key` will return your results.

```
{host}/api/v1/fleet/hosts?order_key=h.created_at&order_direction=desc&after=2022-10-22T20:22:03Z

```
## What can I do if Fleet is slow or unresponsive after enabling a feature?

Depending on your infrastructure capabilities, and the number of hosts enrolled into your Fleet instance, Fleet might be slow or unresponsive after globally enabling a feature like [software inventory](https://fleetdm.com/docs/deploying/configuration#software-inventory).

In those cases, we recommend a slow rollout by partially enabling the feature by teams using the `features` key of the [teams configuration](https://fleetdm.com/docs/using-fleet/configuration-files#teams).

## Why am I getting errors when generating a .msi package on my M1 Mac?

There are many challenges to generating .msi packages on any OS but Windows. Errors will frequently resolve after multiple attempts and we've added retries by default in recent versions of `fleetctl package`.  Package creation is much more reliable on Intel Macs, Linux and Windows.

## Where did Packs go?

Packs are a function of osquery that provide a portable format to import /export queries in and out of platforms like Fleet. These osquery packs still exist, but have been removed from the Fleet UI. Access via API is still available for backwards compatibility.
Within Fleet we've introduced the concept of teams in Fleet premium to target specific groups of hosts, but you can also still use scheduled queries in Fleet free (works like packs) to target all your hosts.

## What happens when I turn off MDM?

In the Fleet UI, you can turn off MDM for a host by selecting **Actions > Turn off MDM** on the **Host details** page.

When you turn off MDM for a host, Fleet removes the enforcement of all macOS settings for that host. Also, the host will stop receiving macOS update reminders via Nudge. Turning MDM off doesn't remove the fleetd agent from the host. To remove the fleetd agent, share [these guided instructions](#how-can-i-uninstall-the-osquery-agent) with the end user.

To enforce macOS settings and send macOS update reminders, the host has to turn MDM back on. To turn MDM on, share [these guided instructions](https://fleetdm.com/docs/using-fleet/mdm-migration-guide#instructions-for-end-users) with the end user. Turning MDM back on for a host requires end user action.

## What does "package root files: heat failed" mean?
We've found this error when you try to build an MSI on Docker 4.17. The underlying issue has been fixed in Docker 4.18, so we recommend upgrading. More information [here](https://github.com/fleetdm/fleet/issues/10700)
