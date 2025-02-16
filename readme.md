# Zorobot 9000

Zorobot (9000) is a multi-model, multi-domain, continuous learning chatbot. It's designed to be easily deployed, easily retrained, and with a pluggable interface to communicate over any number of text messaging apps, sms, slack, facebook, whatsapp, etc. It is implemented as a multi-domain conversational AI service, or rather set of microservces including:
- intent inference engine
- dialogue inference engine
- learning api
- teaching mode

This framework includes a general conversation domain as well as specially traind agents for coordinating calls and meetings, dealing with staffing logistics, website concierge, and an rudimentary "virtual doctor." It supports both passive mode where the user intitiates a conversaton and active mode where conversations can be scheduled for the bot to initiate. (Mainly used by the Virtual Doc.)

# Table of contents

  * [Overview](#zorobot-9000)
  * [Build](#build)
  * [Install](#install)
  * [Deploy](#deploy)
     * [K8s](#k8s)
     * [Docker](#docker)
     * [Metal](#metal)
 * [Multihost](#multihost)
    * [The Manual Way](#themanualway)
    * [Docker Swarm](#dockerswarm)
  * [Run](#run)
  * [Training Times](#training-times)   
  * [Example Domains](#domains)
     * [Concierge](#concierge)
     * [Dinnerbot](#dinnerbot)
     * [Scheduler](#scheduler)
     * [Staffing](#staffing)
     * [Vimbot](#vimbot)
     * [Virtual Doc](#virtualdoc)
  * [Known Issues](#knownissues)   
  * [License](#license)  


### BUILD

Building Zorobot for your architecture is as simple as running:

```
make build
```

This is an alias for `build-arch` which compiles from the source files (found in `lib/c`) to (the) executable objects (which are) found in `lib/ext`.)

The included Makefile also supports options for `build-source` which creates the source code used by build-arch, and `build-local` which, intended for developers, skips the intermediate step and directly compiles the binaries for your local architecture.

## INSTALL

### Installation and Setup

* **Dependencies:**
Zorobot is is a framework, not a library. As such, all dependencies are managed in the environment.

* **Deployment:**
Zorobot is best deployed with Docker, using the provided scripts, though it can of course be installed and run directly on "bare metal." Bare metal installation is not currently implemented, though it's fully supported. Rolling a custom install script just needs to includes niceties such as downloading and linking whichever spaCy model you prefer (Zorobot ships with en large), sourcing your environment and some other housekeeping steps generally handled more efficiently by Docker.

## DEPLOY

### Kubernetes

Helm chart is still being defined and is not shipped with the current "naught-dot" release.

### Docker

Standard Docker deployment builds and runs 3 containers:
* Model Web Service - This exposes the conversational model as a webservice running over a sequence of ports. An initial request is made to retrieve the current port for the active model to faciliate true Continuous Learning with no downtime for retraining.
* Response Server - Business logic is implemented as a separate service to ensure clean separation of business model from conversation logic.
* SMS Bot - Handles SMS integration via Twilio API.

All 3 containers can be run on a single server, or in microservices mode, deployed to different endpoints. These endpoints may be Kubernetes pods, using the forthoming Helm chart. In lieu of K8s, you'll need to initialize a Docker Swarm to deploy each service to a different endpoint.

### Metal

Zorobot runs cleanly on bare metal, which can be used for development. To deploy to a bare metal server for production, you currently have to roll your own install script as it is not included with the 0.1 release.  **\m/**

## MULTIHOST

If you're not using Kubernetes but still want to run each service on its own node, Zorobot can be deployed in multihost mode, either manully, or using Docker Swarm.

#### The manual way:
Given you know the IP addresses for each of your host machines:
1. Add them to the environment (config/env.yml)
2. Copy docker-compose and Docker file to target hosts.
3. Run docker-compose with service name as argument on each respective host.
#### The Swarm way:
1. Designate one host as Swarm manager and run
  `docker swarm init`
This will generate a join-token which your other nodes can use to join the swarm with `docker swarm join`  


## RUN

Assuming you have a working environment definition file in ~~the `env` directory~~ a`.env` file, simply run `docker-compose up`. (This ofc assumes you have root permissions or have added your user to the docker group. I got 99 problems son, but root permissions ain't one.) To start an individual service, specify it as the target of docker-compose:
* NLU and Dialog Server: `docker-compose up dialog`
* Response Server: `docker-composer up reactions`
* SMS Chatbot: `docker-composer up bot`

## TRAINING TIMES

Current training times in the default domain for differing epoch runs are as follows:

  *  55 epochs ~4 minutes
  * 155 epochs ~7 minutes
  * 255 epochs ~11 minutes
  * 555 epochs ~24 minutes

  Zorobot follows the principle that adding more training data, not adding epochs, is the preferred approach for saving significant development time. Having said that, they each solve different, albeit overlapping problems. If you find you're repeatedly adding training data to solve model accuracy issues, you're doing it wrong.

## KNOWN ISSUES / FEATURE REQUESTS

### Bugs

* The startup script (`bot`) doesn't let you run it inteactively in the command line w/o debug mode on. This is something that may be needed for your testing team, most of whom likely won't understand any of the copius debugging information.

* The "scott" bug.
In testing there was one particular instance where the trained LTSM model mis-predicted, namely when the user says "hi forest, it's scott" (with the grammatically correct apostrophe, and in all lower case) to which the bot consistently replies "Super!" instead of the correct response. This happens whether or not the user includes the comma, but does not happen if they misspell it's as "its" or if they capitalize 'Hi', 'Forest' or 'Scott' -- and does not happen for ANY other names tested, only for "scott" when typed in lower case. (The name "scotty" is likewise unaffected and elicits the correct response.) Worth mentioning, we have no one on the team named "Scott" and the expectation is that adding a few epochs will eliminate this bug; it's just an interesting and highly reproducible quirk.

* Role lookup
Not so much a bug as a better implementation, so refactoring; rather than being included in training data, roles should be a lookup (technically making them a categorical prior) which works better with the CAP model here, which leans on computation for availability but bakes consistency in (while partioning is handled across conversational domains.)

* Case sensitivity
Perhaps not a bug proper, but NER case handing is pretty lame. This should really be taken care of transparently, but, at least for cases with only a single token, our model misses obivous entities, for example, not reognizing "Today" in answer to the question "What day?" Where possible, we strive to keep the training set case free (all lower) as well as punctution free, rather than depending on either of those modes for semantics that are present in speech without them. 

### To Do's

* Cleanup Cython build process:
Cython tends to be a very idiosyncratic wapper for cc/gcc etc, it's more or less an overly opinionated transpiler that adds little to the compilation side, unless you just like spending an inordinate amount of time futzing with distutils. Which is just a judgey way of saying it's okay tho it keeps things at the same level of abstraction. If you are only comfortable wth Python, and not C, CPP or compiling, then stick with .pyx, but if Python is just another higher level tool in your tool chest, come play with the big boys.

* Testing Time:
The "Grok" class in the included Kronos module does a pretty suberb job of understanding just about any date and time representation in natural language (currently English only) but does need to be hammer tested for the gazillion ways people talk about dates and times. In testing, it gets 99% of all times and dates, but need to test more, esp. aganst tail cases of the `clean_time_string` method (Cf. t1, t2.) For example, when o'clock is assumed, not specified. (But also handling "o'clock" as a figure of speech.) Most of the date/time issues are low priority as the expectation is we'll be using Duckling for this.

* Duckling integration:
This looks like what we'll be using going forward for date/time intent extraction, mainly bc of everything else you get for free. Duckling is a rules-based engine, not a machine learning model, but seemingly covers a wide bearth of intents and entities. None of them individually are particularly hard (note for example, all date time extraction is currently handled by a single and very compact "kronos" class), but the sheer range of what you get with Duckling makes it a very attrctive option.

* Timezone support: (not implemented)
If someone specifies a preferred time in diff time zone, it will get interpreted as local TZ. This is on hold pending Duckling integration.

* Multi-domain mode
Currently uses giant "domain of doom" for all domains. This will be refactored once core is updated to 2.0. Also, action handling is cross-domain rather than being domain specific. Ideally, actions will be part of domain logic (with a general, default domain for actions which are not specific to a particular domain.)

### Github issue queue

[Tell us](https://github.com/ForestMars/Zorobot/issues) what we haven't thought of!

### LICENSE

Zorobot is released under either the MIT, Apache or GPL license, depending who you ask. Please [contact us](mailto:thearsgroup@gmail.com) for clarification or schedule a call to discuss by sending a text message to us at (415) 712-2019.
