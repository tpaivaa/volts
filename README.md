[![Stand With Ukraine](https://raw.githubusercontent.com/vshymanskyy/StandWithUkraine/main/banner-direct.svg)](https://stand-with-ukraine.pp.ua)
<p align="center">
<img src="docs/images/logo.png" alt="VOLTSLogo" width="200"/>
</p>

**Voip Open Linear Tester Suite**

Functional tests for VoIP systems based on [`voip_patrol`](https://github.com/igorolhovskiy/voip_patrol), [`sipp`](https://github.com/SIPp/sipp)  and [`docker`](https://www.docker.com/)</br>
Some alternative introduction to the system can be found on [DOU](https://dou.ua/forums/topic/38567/)(ukrainian) or in my talk at [Kamailio World 2022](https://youtu.be/9NGu5LpGSMc?t=7727).

# 10'000 ft. view

System is designed to run simple call scenarios, that you usually do with your desk phones.</br>
Scenarios are run one by one from `scenarios` folder in alphabetical order, which could be considered as a limitation, but also allows you to reuse same accounts in a different set of tests. This stands for `Linear` in the name ;)
So, call some destination(s) with one(or more) device(s) and control call arrival on another phone(s).</br>
But wait, there is more. VOLTS also can integrate with your MySQL and/or PostgreSQL databases to write some data there before test and remove it after.</br>
Also it can record (and play, obviously) media during the call and do media check of these files (currently basic via [SoX](https://sox.sourceforge.net/))</br>
It will make, receive calls and configure database. *It's not do transfers at the moment. Sorry. I don't need it*</br></br>

Suite consists of 5 parts, that are running sequentially
1. Preparation - at this part we're transforming templates to real scenarios of `voip_patrol`, `sipp`, `database` and `media_check` using [`Jinja2`](https://jinja.palletsprojects.com/en/3.0.x/) template engine with [jinja2_time](https://github.com/hackebrot/jinja2-time) extension to put some dynamic data based on time.
---
2. Running database scripts. Usually - put some data inside some routing or subscriber data.
3. Running `voip_patrol` or `sipp` scenario.
4. Again running database scripts. Usually - remove data that had been put at the stage 2.
5. Run `media_check` if necessary to analyse obtained media files
---
6. Report - at this part we're analyzing results of previous step reading and interpreting file obtained running steps 2-5. Printing results in a desired way. Table by default.
Steps 2-5 are running sequentially against scenarios files prepared at step 1. One at a time. Again, it's for `Linear`
# Building

Suite is designed to run locally from your Linux PC or Mac (maybe). And of course, `docker` should be installed. It's up to you.</br>
To build, just run `./build.sh`. It would build 5 `docker` images and tag em accordingly.</br>
In a case if `voip_patrol` is updated, you need to rebuild it's container again, you can do it with `./build.sh -r`

# Running

After building, just run
```sh
./run.sh
```
Simple, isn't it? This will run all scenarios found in `scenarios` folder one by one. To run single scenario, run
```sh
./run.sh <scenario_name>
```
or
```sh
./run.sh scenarios/<scenario_name>
```
After running of the suite you can always find a `voip_patrol` presented results in `tmp/output` folder.

But simply run something blindly is boring, so before this, best to do some

# Configuration

We suppose to configure 2 parts here. First, and most complex are

## Scenarios

`VOLTS` scenarios are combined `voip_patrol`/`sipp`, `database` and `media_check` scenarios, that are just being templatized with `Jinja2` style. Mostly done not to repeat some passwords, usernames, domains, etc.</br>
Also, due to using `jinja2-time` extension, it's possible to use dynamic time/date values in your scenarios, for example testing some time-based rules on your PBX. For full documentation on how to use this type of data, please refer to ['jinja2-time'](https://github.com/hackebrot/jinja2-time) documentation.</br>
As you will see below, core for all type of tests are actually `voip_patrol`, others are just helpers around.
### Global config

Values for templates are taken from `scenarios/config.yaml`</br>
One thing to mention here, that vars from `global` section transforms to `c.` (for `config` or `g.` for `global`, `c.` and `.g` are equal) and from `accounts` to `a.` in templates for shorter notation.</br>
There is special name `scenario_name` that is transforming to scenario file name stripped `.xml` extension.</br>
Also all settings from `global` section are inherited to `accounts` section automatically, unless they are defined there explicitly.</br>

`config.yaml`
```yaml
global:
  domain:     '<SOME_DOMAIN>'
  transport:  'tls'
  srtp:       'dtls,sdes,force'
  play_file:  '/voice_ref_files/8000_12s.wav'
databases:
  'kamdb':
    type:       'mysql'
    user:       'kamailiorw'
    password:   'kamailiorwpass'
    base:       'kamailio'
    host:       'mykamailiodb.local'
accounts:
  '88881':
    username:       '88881'
    auth_username:  '88881-1'
    password:       'SuperSecretPass1'
  '88882':
    username:       '88882'
    auth_username:  '88882-67345'
    password:       'SuperSecretPass2'
  '90001':
    username:       '90001'
    auth_username:  '90001'
    password:       'SuperSecretPass3'
```
### VoIP - patrol
To get most of it, please refer to [`voip_patrol`](https://github.com/igorolhovskiy/voip_patrol/voip_patrol) config, but here follows some basic example to show the idea of the templating.</br></br>
**Make a register**
```xml
<config>
    <section type="voip_patrol">
        <actions>
            <action type="register" label="Register {{ a.88881.label }}"
                transport="{{ a.88881.transport }}"
                <!-- Account parameter is more used in receive call on this account later -->
                account="{{ a.88881.label }}"
                <!-- username would be a part of AOR - <sip:username@realm> -->
                username="{{ a.88881.username }}"
                <!-- auth_username would be used in WWW-Authorize procedure -->
                auth_username="{{ a.88881.auth_username }}"
                password="{{ a.88881.password }}"
                registrar="{{ c.domain }}"
                realm="{{ a.88881.domain }}"
                <!-- We are expecting get 200 code here, so REGISTER is successfull -->
                expected_cause_code="200"
            />
            <!-- Just wait 2 sec for all timeouts -->
            <action type="wait" complete="true" ms="2000"/>
        </actions>
    </section>
</config>
```
### SIPP
You can just add your existing SIPP scenarios mainly unchanged and they would be run with the following command:
```sh
sipp <target> -sf <scenario.xml> -m 1 -mp <random_port> -i <container_ip>
```
**Send an OPTIONS**
```xml
<config>
    <section type="sipp">
        <actions>
            <action transport="{{ c.transport }}" target="{{ c.domain }}">
                <scenario name="Options">
                    <send>
                        <![CDATA[
                OPTIONS sip:check_server_health@{{ c.domain }} SIP/2.0
                Via: SIP/2.0/[transport] [local_ip]:[local_port];branch=[branch]
                Max-Forwards: 70
                To: <sip:check_server_health@{{ c.domain }}>
                From: sipp <sip:check_server_health@[local_ip]:[local_port]>;tag=[call_number]
                Call-ID: [call_id]
                CSeq: 1 OPTIONS
                Contact: <sip:check_server_health@[local_ip]:[local_port]>
                Accept: application/sdp
                Content-Length: 0
                ]]>
                    </send>
                    <recv response="200" timeout="200"/>
                </scenario>

            </action>
        </actions>
    </section>
</config>
```
| Attribute | Description |
| --- | --- |
| transport | Actual transport SIPP will use. Values are `udp`(default), `tcp` or `tls`. |
| target | What you usually specify as target when running standalone SIPP. If transport is TLS, if no port is pecified, `5061` is appended by default. |
### Database
Database config is also done in XML, section `database`. We have 2 `stage`s of database scripts.
| Stage | Description |
| --- | --- |
| `pre` | Lauched before running `voip_patrol`. Usually stage to put some accounts data, routing, etc. |
| `post` | Obviously, running after `voip_patrol`. For cleanup data inserted in `pre` stage. |


So, inside `database` action you specify tables you're working with. Each `table` secton have 4 attributes.
| Attribute | Description |
| --- | --- |
| `name` | Actually name of table we're working with. |
| `type` | Could be `insert`, `replace` and `delete`. Forming actually `INSERT`, `REPLACE` and `DELETE` SQL statements for database. |
| `continue_on_error` | Optional. Em.. ignore errors on preformed actions and continue no matter what. By default database actions will be stopped after encountering first error. |
| `cleanup_after_test` | Optional. Allows you not to write explicit `post` stage for your `insert` types. Will automatically form `delete` type on `post` stage for all `insert` (but not `replace`) that were declared on `pre` stage |
**Make a register with database**
```xml
<!-- Test simple register -->
<config>
    <section type="database">
        <actions>
             <!-- "kamdb" here is referring to entity in "databases" from config.yaml. -->
            <action database="kamdb" stage="pre">
                <!-- what data are we gonna insert into "subscriber" table? -->
                <table name="subscriber" type="insert" cleanup_after_test="true">
                    <field name="username" value="{{ a.88881.username }}"/>
                    <field name="domain" value="{{ c.domain }}"/>
                    <field name="password" value="{{ a.88881.password }}"/>
                </table>
            </action>
        </actions>
    </section>
    <section type="voip_patrol">
        <actions>
            <action type="register" label="Register {{ a.88881.label }}"
                transport="{{ a.88881.transport }}"
                <!-- Account parameter is more used in receive call on this account later -->
                account="{{ a.88881.label }}"
                <!-- username would be a part of AOR - <sip:username@realm> -->
                username="{{ a.88881.username }}"
                <!-- auth_username would be used in WWW-Authorize procedure -->
                auth_username="{{ a.88881.auth_username }}"
                password="{{ a.88881.password }}"
                registrar="{{ c.domain }}"
                realm="{{ a.88881.domain }}"
                <!-- We are expecting get 200 code here, so REGISTER is successfull -->
                expected_cause_code="200"
            />
            <!-- Just wait 2 sec for all timeouts -->
            <action type="wait" complete="true" ms="2000"/>
        </actions>
    </section>
</config>
```
### Media check
You can analyze calls recording with various media tools. *Currently only SoX is supported.*</br>
Media check is also described in XML
| Attribute | Description |
| --- | --- |
| `type` | Mandatory. Media check test to be preformed. Currently only `sox`. |
| `file` | Mandatory. Path to file to check. Have to be aligned with `record` in one of `voip_patrol` actions. Best to have it with distinct names and currently with `/output/` prefix due to container interconnection (will be fixed later), see example below for better picture |
| `delete_after` | Do we delete file after media check? `yes`/`no`. `yes` by default |
| `print_debug` | Print debug info on file on console while testing. Usefult for adjusting `filter` parameters. `yes`/`no`. `no` by default |
| `sox_filter` | Used if `type` is `sox`. Semicolon-separated expressions to test values obtained by SoX utility with the given file. Usually to check some float values like lenght or amplitude. See below more detailed description |

#### SoX media check
Within this check parameters from the `file` are collected by `sox` utility, more presice - `sox --i <file>`, `sox <file> -n stat`, `sox <file> -n stats`.</br>
In a `sox_filter` attribute you can write a string to check some given values against collected parameters. All filters expressions should be true to test pass. Best to be explained on the example</br>
```
sox_filter="length s -ge 10; length s -le 11"
```
Here we have 1 parameter - `length s` that should be grater than or equal 10 and less than or equal 11. `length s` is actually one of result parameters that are obtained by `sox <file> -n stats`.</br>
Point, here is used not traditional `<=` style notation, but `bash` (`-eq` is `==`, `-lt` is `<`, `-gt` is `>`, `-le` is `<=`, `-ge` is `>=`, `-ne` is `!=`) style comparsion operators.
This is done due to traditional comparsion symbols (`<`,`>`) are part of XML notation</br>
Getting parameters names is simple - they are converted from `sox` outputs, example:
```
# sox 8000_12s.wav -n stats
DC offset  -0.000078    -> 'dc offset'     float
Min level  -0.321594    -> 'min level'     float
Max level   0.430359    -> 'max level'     float
Pk lev dB      -7.32    -> 'pk lev db'     float
RMS lev dB    -29.63    -> 'rms lev db'    float
RMS Pk dB     -16.78    -> 'rms pk db'     float
RMS Tr dB     -74.77    ...
Crest factor   13.04    ...
Flat factor     0.00    ...
Pk count           2    ...
Bit-depth      15/16    -> 'bit-depth'     string
Num samples     531k    -> 'num samples'   string
Length s      12.034    -> 'length s'      float
Scale max   1.000000    ...
Window s       0.050    ...
```
Here all parameter names are lowercased. One more example with the data above:
```
sox_filter="length s -ge 11; crest factor -lt 10; bit-depth -eq 15/16"
```
All number-like values are automatically threated as numbers and you can apply `-lt`, `-ge` type of comparsions.</br></br>
**Make a call to echo number and analyse outcome**
```xml
<!-- Call echo service and name sure receive an answer -->
<config>
    <section type="voip_patrol">
        <actions>
            <action type="codec" disable="all"/>
            <action type="codec" enable="pcma" priority="250"/>
            <action type="codec" enable="pcmu" priority="249"/>
            <action type="call" label="Call to 11111 (echo)"
                transport="{{ a.88881.transport }}"
                <!-- We are expecting answer here -->
                expected_cause_code="200"
                caller="{{ a.88881.label }}@{{ c.domain }}"
                <!-- Setting R-URI -->
                callee="11111@{{ a.88881.domain }}"
                from="sip:{{ a.88881.label }}@{{ c.domain }}"
                to_uri="11111@{{ c.domain }}"
                max_duration="20" hangup="10"
                auth_username="{{ a.88881.username }}"
                password="{{ a.88881.password }}"
                realm="{{ c.domain }}"
                play="{{ c.play_file }}"
                rtp_stats="true"
                srtp="{{ a.88881.srtp }}"
                <!-- We heed to record file on answer. To analyze it below now it MUST be with "/output/" path prefix -->
                record="/output/{{ scenario_name }}.wav"
            />
            <action type="wait" complete="true" ms="30000"/>
        </actions>
    </section>
    <section type="media_check">
        <actions>
            <action type="sox"
                sox_filter="length s -ge 10; length s -le 11"
                <!-- File name is same as in "record" attribute in "call" action above. Now it MUST be with "/output/" path prefix -->
                file="/output/{{ scenario_name }}.wav"
            />
        </actions>
    </section>
</config>

```


## `run.sh` script

Not that much to configure here, mostly you'll be interested in setting environement variables at the start of the script
| Variable name | Description |
| --- | --- |
|`REPORT_TYPE` | Actually, report type, that would be provided at the end. </br>`table` - print results in table, only failed tests are pritend. </br>`json` - print results in JSON format, only failed tests are pritend. </br>`table_full`, `json_full` - prints results in table or JSON respectively, but print full info on tests passed
| `LOG_LEVEL` | `voip_patrol`/`sipp` log level on the console. In a case of `sipp` scenarios debug it's useful to have this value equal to 3 |

# Results

As a results, you will have table like this.
```
+---------------------------------------+-----------------------------------------------------------+------+----------+-------+--------+------------------+
|                              Scenario |                                               VoIP Patrol | SIPP | Database | Media | Status |             Text |
+---------------------------------------+-----------------------------------------------------------+------+----------+-------+--------+------------------+
|                           01-register |                                                      PASS |  N/A |      N/A |   N/A |   PASS |  Scenario passed |
|                                       |                                      Register 88881       |      |          |       |   PASS | Main test passed |
|                          02-call-echo |                                                      PASS |  N/A |      N/A |   N/A |   PASS |  Scenario passed |
|                                       |                                      Call to 11111 (echo) |      |          |       |   PASS | Main test passed |

....
|            51-call-echo-media-control |                                                      PASS |  N/A |      N/A |  PASS |   PASS |  Scenario passed |
|                                       |                                      Call to 11111 (echo) |      |          |       |   PASS | Main test passed |
| 52-delayed-call-forward-unconditional |                                                      PASS |  N/A |     PASS |   N/A |   PASS |  Scenario passed |
|                                       |                                      Register 90012       |      |          |       |   PASS | Main test passed |
|                                       |                                      Register 90013       |      |          |       |   PASS | Main test passed |
|                                       |                      Receive call on 90012 and not answer |      |          |       |   PASS |    Call canceled |
|                                       |   Call from 90011 to 90012 (delay forward 25 sec) ->90013 |      |          |       |   PASS | Main test passed |
|                                       |                       Receive call on 90013       finally |      |          |       |   PASS | Main test passed |
|                53-server-check-health |                                                       N/A | PASS |      N/A |   N/A |   PASS | SIPP test passed |
+---------------------------------------+-----------------------------------------------------------+------+----------+-------+--------+------------------+

Scenarios ['49-teams-follow-forward', '50-team-no-answer-forward'] are failed!
```
That means your system is not OK, or something need to be tuned with the tests.</br>
Not really much to describe here, just read info on the console

# Scenario Examples
Examples shown in this section can duplicate examples from the section above. For more examples, refer to the `scenarios` folder.</br>
### Make a successful register
 * Here we assume that account data is known to our PBX.
```xml
<config>
    <section type="voip_patrol">
        <actions>
            <action type="register" label="Register {{ a.88881.label }}"
                transport="{{ a.88881.transport }}"
                <!-- Account parameter is more used in receive call on this account later -->
                account="{{ a.88881.label }}"
                <!-- username would be a part of AOR - <sip:username@realm> -->
                username="{{ a.88881.username }}"
                <!-- auth_username would be used in WWW-Authorize procedure -->
                auth_username="{{ a.88881.auth_username }}"
                password="{{ a.88881.password }}"
                registrar="{{ c.domain }}"
                realm="{{ a.88881.domain }}"
                <!-- We are expecting get 200 code here, so REGISTER is successfull -->
                expected_cause_code="200"
            />
            <!-- Just wait 2 sec for all timeouts -->
            <action type="wait" complete="true" ms="2000"/>
        </actions>
    </section>
</config>
```
* .. but what if we need to add account data to PBX dynamically? Assume, that we have Kamailio as a PBX here.
```xml
<!-- Test simple register -->
<config>
    <section type="database">
        <actions>
             <!-- "kamdb" here is referring to entity in "databases" from config.yaml. "stage" is explained a bit below -->
            <action database="kamdb" stage="pre">
                <!-- what data are we gonna insert into "subscriber" table? -->
                <table name="subscriber" type="insert" cleanup_after_test="true">
                    <field name="username" value="{{ a.88881.username }}"/>
                    <field name="domain" value="{{ c.domain }}"/>
                    <field name="password" value="{{ a.88881.password }}"/>
                </table>
            </action>
        </actions>
    </section>
    <section type="voip_patrol">
        <actions>
            <action type="register" label="Register {{ a.88881.label }}"
                transport="{{ a.88881.transport }}"
                <!-- Account parameter is more used in receive call on this account later -->
                account="{{ a.88881.label }}"
                <!-- username would be a part of AOR - <sip:username@realm> -->
                username="{{ a.88881.username }}"
                <!-- auth_username would be used in WWW-Authorize procedure -->
                auth_username="{{ a.88881.auth_username }}"
                password="{{ a.88881.password }}"
                registrar="{{ c.domain }}"
                realm="{{ a.88881.domain }}"
                <!-- We are expecting get 200 code here, so REGISTER is successfull -->
                expected_cause_code="200"
            />
            <!-- Just wait 2 sec for all timeouts -->
            <action type="wait" complete="true" ms="2000"/>
        </actions>
    </section>
</config>
```

### Expect fail on register
We're deleting data from database and restoring it afterwards.
```xml
<config>
    <section type="database">
        <actions>
            <action database="kamdb" stage="pre">
                <table name="subscriber" type="delete" cleanup_after_test="true">
                    <field name="username" value="{{ a.88881.username }}"/>
                    <field name="domain" value="{{ c.domain }}"/>
                    <field name="password" value="{{ a.88881.password }}"/>
                </table>
            </action>
        </actions>
    </section>
    <section type="voip_patrol">
        <actions>
            <action type="register" label="Register {{ a.88881.label }}"
                transport="{{ a.88881.transport }}"
                account="{{ a.88881.label }}"
                username="{{ a.88881.username }}"
                auth_username="{{ a.88881.auth_username }}"
                password="{{ a.88881.password }}"
                registrar="{{ c.domain }}"
                realm="{{ a.88881.domain }}"
                <!-- We are expecting get 407 code here, maybe your registrar sending 401 or 403 code. So - adjust it here. -->
                expected_cause_code="407"
            />
            <action type="wait" complete="true" ms="2000"/>
        </actions>
    /section>
</config>
```

### Simple call scenario
Register with 1 account and make a call from `90001` to `88881`. Max wait time to answer - 15 sec, duration of connected call - 10 sec.</br>
Point, we don't register account `90001` here, as we're not receiving a calls on it, just need to provide credentials on INVITE.</br>
Also trick, `match_account` in `accept` perfectly links with `account` in `register`.
```xml
<config>
    <actions>
        <!-- As we're using call functionality here - define list of codecs -->
        <action type="codec" disable="all"/>
        <action type="codec" enable="pcma" priority="250"/>
        <action type="codec" enable="pcmu" priority="249"/>
        <action type="codec" enable="opus" priority="248"/>
        <action type="register" label="Register {{ a.88881.label }}"
            transport="{{ a.88881.transport }}"
            account="{{ a.88881.label }}"
            username="{{ a.88881.username }}"
            auth_username="{{ a.88881.auth_username }}"
            password="{{ a.88881.password }}"
            registrar="{{ c.domain }}"
            realm="{{ c.domain }}"
            expected_cause_code="200"
            <!-- Make sure we are using SRTP on a call receive. This is done here as accounts are created before accept(answer) action -->
            srtp="{{ a.88881.srtp }}"
        />
        <action type="wait" complete="true" ms="2000"/>
        <action type="accept" label="Receive call on {{ a.88881.label }}"
            <!-- This is not a load test - so only 1 call is expected -->
            call_count="1"
            <!-- Make sure we're received a call on a previously registered account -->
            match_account="{{ a.88881.label }}"
            <!-- Hangup in 10 seconds after answer -->
            hangup="10"
            <!-- Send back "200 OK" -->
            code="200" reason="OK"
            transport="{{ a.88881.transport }}"
            <!-- Make sure we're using SRTP -->
            srtp="{{ a.88881.srtp }}"
            <!-- Play some file back to gather RTCP stats in report -->
            play="{{ c.play_file }}"
        />
        <action type="call" label="Call {{ a.90001.label }} -> {{ a.88881.label }}"
            transport="tls"
            <!-- We are waiting for an answer -->
            expected_cause_code="200"
            caller="{{ a.90001.label }}@{{ c.domain }}"
            callee="{{ a.88881.label }}@{{ c.domain }}"
            from="sip:{{ a.90001.label }}@{{ c.domain }}"
            to_uri="{{ a.88881.label }}@{{ c.domain }}"
            max_duration="20" hangup="10"
            <!-- We're specifying all auth data here for INVITE -->
            auth_username="{{ a.90001.username }}"
            password="{{ a.90001.password }}"
            realm="{{ c.domain }}"
            rtp_stats="true"
            max_ring_duration="15"
            srtp="{{ a.90001.srtp }}"
            play="{{ c.play_file }}"
        />
        <action type="wait" complete="true" ms="30000"/>
    </actions>
</config>
```
### Advanced call scenario
Register with 2 accounts and call from third one, not answer on 1st and make sure we receive call on second. So, your PBX should be configured to make a Forward-No-Answer from `88881` to `88882`.</br>
Also make sure, that on `88882` we got the call from `90001` (based on CallerID).
```xml
<config>
    <actions>
        <action type="codec" disable="all"/>
        <action type="codec" enable="pcma" priority="250"/>
        <action type="codec" enable="pcmu" priority="249"/>
        <action type="codec" enable="opus" priority="248"/>
        <action type="register" label="Register {{ a.88881.label }}"
            transport="{{ a.88881.transport }}"
            account="{{ a.88881.label }}"
            username="{{ a.88881.username }}"
            auth_username="{{ a.88881.auth_username }}"
            password="{{ a.88881.password }}"
            registrar="{{ c.domain }}"
            realm="{{ c.domain }}"
            expected_cause_code="200"
            srtp="{{ a.88881.srtp }}"
        />
        <action type="register" label="Register {{ a.88882.label }}"
            transport="{{ a.88882.transport }}"
            account="{{ a.88882.label }}"
            username="{{ a.88882.username }}"
            auth_username="{{ a.88882.auth_username }}"
            password="{{ a.88882.password }}"
            registrar="{{ c.domain }}"
            realm="{{ c.domain }}"
            expected_cause_code="200"
            srtp="{{ a.88882.srtp }}"
        />
        <action type="wait" complete="true" ms="2000"/>
        <action type="call" label="Call from 90001 to 88881->88882"
            transport="{{ a.90001.transport }}"
            expected_cause_code="200"
            caller="{{ a.90001.label }}@{{ c.domain }}"
            callee="88881@{{ c.domain }}"
            from="sip:{{ a.90001.label }}@{{ c.domain }}"
            to_uri="88881@{{ c.domain }}"
            max_duration="20" hangup="10"
            auth_username="{{ a.90001.username }}"
            password="{{ a.90001.password }}"
            realm="{{ c.domain }}"
            rtp_stats="true"
            <!-- Set some high ring timeout, so delayed forward will happen -->
            max_ring_duration="60"
            srtp="{{ a.90001.srtp }}"
            play="{{ c.play_file }}"
        />
        <action type="accept" label="Receive call on {{ a.88881.label }}"
            match_account="{{ a.88881.label }}"
            call_count="1"
            hangup="10"
            ring_duration="30"
            <!-- We're expecting a CANCEL here. And it's not optional -->
            cancel="force"
            transport="{{ a.88881.transport }}"
            srtp="{{ a.88881.srtp }}"
        />
        <action type="accept" label="Receive call on {{ a.88882.label }}"
            match_account="{{ a.88882.label }}"
            call_count="1"
            hangup="10"
            code="200" reason="OK"
            transport="{{ a.88882.transport }}"
            srtp="{{ a.88882.srtp }}"
            play="{{ c.play_file }}">
            <!-- Check that From header matching what we need. This way we can control CallerID. Adjust domain (and whole regex) accordingly -->
            <check-header name="From" regex="^.*sip:{{ a.90001.label }}@example\.com>.*$"/>
        </action>
        <action type="wait" complete="true" ms="20000"/>
    </actions>
</config>
```

### Advanced call scenario - 2. Now with databases.
Schema - Kamailio subscriber and than we have Asterisk behind as PBX.

`config.yaml`
```yaml
global:
  domain:           '<SOME_DOMAIN>'
  transport:        'tls'
  srtp:             'dtls,sdes,force'
  play_file:        '/voice_ref_files/8000_2m30.wav'
  asterisk_context: 'default'
databases:
  'kamdb':
    type:       'mysql'
    user:       'kamailiorw'
    password:   'kamailiorwpass'
    base:       'kamailio'
    host:       'mykamailiodb.local'
  'astdb':
    type:       'pgsql'
    user:       'asteriskrw'
    password:   'asteriskrwpass'
    base:       'asterisk'
    host:       'myasteriskdb.local'
accounts:
  '90011':
    username: '90011'
    password: 'SuperSecretPass1'
    ha1:      'SuperSecretHA1'
  '90012':
    username: '90012'
    password: 'SuperSecretPass2'
    ha1:      'SuperSecretHA2'
```
And now we need to populate all databases and make a call!
```xml
<!-- Register with 90012 and receive a call from 90011 -->
<config>
    <section type="database">
        <actions>
        <!-- add subscribers to Kamailio -->
            <action database="kamdb" stage="pre">
                <table name="subscriber" type="insert"  cleanup_after_test="true">
                    <field name="username" value="{{ a.90011.username }}"/>
                    <field name="domain" value="{{ c.domain }}"/>
                    <field name="ha1" value="{{ a.90011.ha1 }}"/>
                    <!-- here password due to ha1 is useless, so we can put some data based on jinja2_time.TimeExtension (https://github.com/hackebrot/jinja2-time) -->
                    <field name="password" value="{% now 'local' %}"/>
                </table>
                <table name="subscriber" type="insert"  cleanup_after_test="true">
                    <field name="username" value="{{ a.90012.username }}"/>
                    <field name="domain" value="{{ c.domain }}"/>
                    <field name="ha1" value="{{ a.90012.ha1 }}"/>
                    <field name="password" value="{% now 'local' + 'days=1', '%D' %}"/>
                </table>
            </action>
            <!-- add endpoints and aors to Asterisk -->
            <action database="astdb" stage="pre">
                <table name="ps_endpoints" type="insert"  cleanup_after_test="true">
                    <field name="id" value="{{ a.90011.label }}"/>
                    <field name="transport" value="transport-udp"/>
                    <field name="aors" value="{{ a.90011.label }}"/>
                    <field name="context" value="{{ c.asterisk_context }}"/>
                    <field name="disallow" value="all"/>
                    <field name="allow" value="!all,opus,alaw"/>
                    <field name="direct_media" value="no"/>
                    <field name="ice_support" value="no"/>
                    <field name="rtp_timeout" value="3600"/>
                </table>
                <table name="ps_aors" type="insert"  cleanup_after_test="true">
                    <field name="id" value="{{ a.90011.label }}"/>
                    <field name="contact" value="sip:{{ a.90011.label }}@{{ c.domain }}:5060"/>
                </table>
                <table name="ps_endpoints" type="insert"  cleanup_after_test="true">
                    <field name="id" value="{{ a.90012.label }}"/>
                    <field name="transport" value="transport-udp"/>
                    <field name="aors" value="{{ a.90012.label }}"/>
                    <field name="context" value="{{ c.asterisk_context }}"/>
                    <field name="disallow" value="all"/>
                    <field name="allow" value="!all,opus,alaw"/>
                    <field name="direct_media" value="no"/>
                    <field name="ice_support" value="no"/>
                    <field name="rtp_timeout" value="3600"/>
                </table>
                <table name="ps_aors" type="insert"  cleanup_after_test="true">
                    <field name="id" value="{{ a.90012.label }}"/>
                    <field name="contact" value="sip:{{ a.90012.label }}@{{ c.domain }}:5060"/>
                </table>
            </action>
        </actions>
    </section>
    <section type="voip_patrol">
    <!-- Make a call from one endpoint to other -->
        <actions>
            <action type="codec" disable="all"/>
            <action type="codec" enable="pcma" priority="250"/>
            <action type="codec" enable="pcmu" priority="249"/>
            <action type="codec" enable="opus" priority="248"/>
            <action type="register" label="Register {{ a.90012.label }}"
                transport="{{ a.90012.transport }}"
                account="{{ a.90012.username }}"
                username="{{ a.90012.label }}"
                auth_username="{{ a.90012.username }}"
                password="{{ a.90012.password }}"
                registrar="{{ c.domain }}"
                realm="{{ c.domain }}"
                expected_cause_code="200"
                srtp="{{ a.90012.srtp }}"
            />
            <action type="wait" complete="true" ms="2000"/>
            <action type="accept" label="Receive call on {{ a.90012.label }} from {{ a.90011.label }}"
                call_count="1"
                match_account="{{ a.90012.username }}"
                hangup="10"
                code="200" reason="OK"
                transport="{{ a.90012.transport }}"
                srtp="{{ a.90012.srtp }}"
                play="{{ c.play_file }}">
                <check-header name="From" regex="^.*sip:{{ a.90011.label }}@.*$"/>
            </action>
            <action type="call" label="Call {{ a.90011.label }} -> {{ a.90012.label }}"
                transport="tls"
                expected_cause_code="200"
                caller="{{ a.90011.label }}@{{ c.domain }}"
                callee="{{ a.90012.label }}@{{ c.domain }}"
                from="sip:{{ a.90011.label }}@{{ c.domain }}"
                to_uri="{{ a.90012.label }}@{{ c.domain }}"
                max_duration="20" hangup="10"
                auth_username="{{ a.90011.username }}"
                password="{{ a.90011.password }}"
                realm="{{ c.domain }}"
                rtp_stats="true"
                max_ring_duration="15"
                srtp="{{ a.90011.srtp }}"
                play="{{ c.play_file }}"
            />
            <action type="wait" complete="true" ms="30000"/>
        </actions>
    </section>
</config>
```
### Adding media check.

```xml
<!-- Call echo service and name sure receive an answer -->
<config>
    <section type="voip_patrol">
        <actions>
            <action type="codec" disable="all"/>
            <action type="codec" enable="pcma" priority="250"/>
            <action type="codec" enable="pcmu" priority="249"/>
            <action type="call" label="Call to 11111 (echo)"
                transport="{{ a.88881.transport }}"
                <!-- We are expecting answer here -->
                expected_cause_code="200"
                caller="{{ a.88881.label }}@{{ c.domain }}"
                <!-- Setting R-URI -->
                callee="11111@{{ a.88881.domain }}"
                from="sip:{{ a.88881.label }}@{{ c.domain }}"
                to_uri="11111@{{ c.domain }}"
                max_duration="20" hangup="10"
                auth_username="{{ a.88881.username }}"
                password="{{ a.88881.password }}"
                realm="{{ c.domain }}"
                play="{{ c.play_file }}"
                rtp_stats="true"
                srtp="{{ a.88881.srtp }}"
                <!-- We heed to record file on answer. To analyze it below now it MUST be with "/output/" path prefix -->
                record="/output/{{ scenario_name }}.wav"
            />
            <action type="wait" complete="true" ms="30000"/>
        </actions>
    </section>
    <section type="media_check">
        <actions>
            <action type="sox"
                <!-- We are testing that the outcome of recorded file is between 10 and 11 seconds and checking amplitude -->
                sox_filter="length s -ge 10; length s -le 11; maximum amplitude -ge 0.9; minimum amplitude -le -0.5"
                <!-- File name is same as in "record" attribute in "call" action above. Now it MUST be with "/output/" path prefix -->
                file="/output/{{ scenario_name }}.wav"
            />
        </actions>
    </section>
</config>

```
### Running SIPP tests
```xml
<config>
    <section type="sipp">
        <actions>
            <action transport="{{ c.transport }}" target="{{ c.domain }}">
                <!-- sipp {{ c.domain }}:5061 -sf scen.xml -m 1 -r 1 -t ln -tls_cert /etc/ssl/certs/ssl-cert-snakeoil.pem -tls_key /etc/ssl/private/ssl-cert-snakeoil.key -->
                <scenario name="Options">
                    <send>
                        <![CDATA[
                OPTIONS sip:check_server_health@{{ c.domain }} SIP/2.0
                Via: SIP/2.0/[transport] [local_ip]:[local_port];branch=[branch]
                Max-Forwards: 70
                To: <sip:check_server_health@{{ c.domain }}>
                From: sipp <sip:check_server_health@[local_ip]:[local_port]>;tag=[call_number]
                Call-ID: [call_id]
                CSeq: 1 OPTIONS
                Contact: <sip:check_server_health@[local_ip]:[local_port]>
                Accept: application/sdp
                Content-Length: 0
                ]]>
                    </send>
                    <recv response="200" timeout="200"/>
                </scenario>

            </action>
        </actions>
    </section>
</config>
```
