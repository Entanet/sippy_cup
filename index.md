---
layout: default
---

# Sippy Cup

* Table of Contents
{:toc}


## Overview {#overview}


### The Problem {#the_problem}

Load testing voice systems, and voice applications in particular, is tricky.  While several commercial tools exist, there is really only one tool in the Open Source world that is good at efficiently generating SIP load: [SIPp](http://sipp.sourceforge.net/).  While SIPp does a good job of generating load, it is somewhat clumsy to use, due to a verbose XML format for scenarios, a confusing set of command line parameters, and worst of all, a lack of tools to create media needed to interact with voice applications.

The last problem is especially tricky: Imagine you want to load test an IVR. Testing requires:


* calling a test number
* waiting a certain amount of time
* sending some DTMF
* waiting some more
* sending more DTMF
* etc....

To test this with SIPp you need a PCAP file that contains the properly timed DTMF interactions. Since there is no tool to create this media, it is usually necessary to call into the system and record the PCAP, isolate the RTP from the captured packets with something like Wireshark, then connect the pcap file into the SIPp scenario.  This process is time consuming and error prone, meaning that testing isn't done as often as it should.

SippyCup aims to help solve these problems.

### The Solution {#the_solution}

Sippy Cup is a tool to generate [SIPp](http://sipp.sourceforge.net/) load test profiles and the corresponding media in PCAP format. The goal is to take an input document that describes a load test in a very simple way (call this number, wait this many seconds, send this digit, wait a few more seconds, etc).  The ideas are taken from [LoadBot](https://github.com/mojolingo/ahn-loadbot), but the goal is for a more performant load generating tool with no dependency on Asterisk.

## Requirements {#requirements}

SippyCup relies on the following to generate scenarios and the associated media PCAP files:

* Ruby 1.9.3 (2.0.0 NOT YET SUPPORTED; see [PacketFu Issue #28](https://github.com/todb/packetfu/issues/28))
* [SIPp](http://sipp.sourceforge.net/) - Download from http://sourceforge.net/projects/sipp/files/
* "root" user access via sudo: needed to run SIPp so it can bind to raw network sockets

## Installation {#installation}

If you do not have Ruby 1.9.3 available (check using `ruby --version`), we recommend installing Ruby with [RVM](http://rvm.io)

Once Ruby is installed, install SippyCup:

```
gem install sippy_cup
```

Now you can start creating scenario files like in the examples below.

## Examples {#examples}

{% highlight ruby %}
require 'sippy_cup'

scenario = SippyCup::Scenario.new 'Sippy Cup', source: '192.168.5.5:10001', destination: '10.10.0.3:19995' do |s|
  s.invite
  s.wait_for_answer
  s.ack_answer

  s.sleep 3
  s.send_digits '3125551234'
  s.sleep 5
  s.send_digits '#'

  s.receive_bye
  s.ack_bye
end

# Create the scenario XML, PCAP media, and YAML options. File will be named after the scenario name, in our case:
# * sippy_cup.xml
# * sippy_cup.yml
# * sippy_cup.pcap
scenario.compile!
{% endhighlight %}

The above code can either be executed as a standalone Ruby script and run with SIPp, or it can be compiled and run using rake tasks by inserting the following code into your Rakefile:

{% highlight ruby %}
require 'sippy_cup/tasks'
{% endhighlight %}

Then running the rake task `rake sippy_cup:compile[sippy_cup.rb]` 

And finally running `rake sippy_cup:run[sippy_cup.yml]` to execute the scenario.

## Customizing Scenarios {#customizing_scenarios}

### Alternate Output File Path

Don't want your scenario to end up in the same directory as your script? Need the filename to be different than the scenario name? No problem! Try:

{% highlight ruby %}
my_opts = { source: '192.168.5.5:10001', destination: '10.10.0.3:19995', filename: '/path/to/somewhere' }
s = SippyCup::Scenario.new 'SippyCup', my_opts do
  # scenario statements here...
end
{% endhighlight %}

This will create the files `somewhere.xml`, `somewhere.pcap`, and `somewhere.yml` in the `/path/to/` directory.

### Customizing the Test Run

By default, sippy cup will automatically generate a YAML file with the following contents:

{% highlight yaml %}
---
:source: 127.0.0.1
:destination: 127.0.0.1
:scenario: /path/to/scenario.xml
:max_concurrent: 10
:calls_per_second: 5
:number_of_calls: 20
{% endhighlight %}

Each parameter has an impact on the test, and may either be changed once the YAML file is generated or specified in the options hash for <code>SippyCup::Scenario.new</code>. In addition to the default parameters, some additional parameters can be set:

<dl>
<dt>:source_port:</dt>
  <dd>The local port from which to originate SIP traffic. This defaults to port 8836</dd>

  <dt>:stats_file:</dt>
  <dd>Path to a file where call statistics will be stored in a CSV format, defaults to not storing stats</dd>

  <dt>:stats_interval</dt>
  <dd>Frequency (in seconds) of statistics collections. Defaults to 10. Has no effect unless :stats_file is also specified</dd>

  <dt>:sip_user:</dt>
  <dd>SIP username to use. Defaults to "1" (as in 1@127.0.0.1)</dd>

  <dt>:full_sipp_output:</dt>
  <dd>By default, SippyCup will hide SIPp's command line output while running a scenario. Set this parameter to `true` to see full command line output</dd>
</dl>

### Additional SIPp Scenario Attributes

With Sippy Cup, you can add additional attributes to each step of the scenario:

{% highlight ruby %}
#This limits the amount of time the server has to reply to an invite (3 seconds)
s.receive_answer timeout: 3000

#You can override the default 'optional' parameters
s.receive_ringing optional: false
s.receive_answer optional: true

#Let's combine multiple attributes...
s.receive_answer timeout: 3000, crlf: true
{% endhighlight %}

For more information on possible attributes, visit the [SIPp Documentation](http://sipp.sourceforge.net/doc/reference.html).