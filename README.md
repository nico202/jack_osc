# Open Sound Control (OSC) via Jack

## About
MIDI does not fit all our needs as a musical event system any more. This comes as no surprise as it is from the 80's of the last century. People therefore started to come up and use more capable alternative musical event systems like e.g. [OSC](http://opensoundcontrol.org/spec-1_0) or [LV2 Atoms](http://lv2plug.in/ns/ext/atom/).

OSC today is mainly used via network sockets (UDP/TCP). For a modern musical event system however there is the need for sample-accurate, low-latency event routing. We already have our beloved [Jack](http://jackaudio.org) Audio Connection Kit for exactly this purpose. Sadly, it officially supports only MIDI as its sole musical event.

**This is a workaround for Jack to support routing sample-accurate OSC messages via Jack MIDI ports as discussed at [LAC2014](http://lac.linuxaudio.org/2014/).**

Get the poster in question [here](http://lac.linuxaudio.org/2014/download/routing_OSC_via_vanilla_JACK.pdf).

## Jack MIDI

From [jack/midiport.h](https://github.com/jackaudio/headers/blob/master/midiport.h) we learn:

	typedef unsigned char jack_midi_data_t;

	typedef struct _jack_midi_event
	{
		jack_nframes_t    time;   // Sample index at which event is valid 
		size_t            size;   // Number of bytes of data in a buffer
		jack_midi_data_t *buffer; // Raw MIDI data
	} jack_midi_event_t;

When Jack routes MIDI, the raw 8-bit MIDI bytes are embedded in the structure's buffer.

Example MIDI NoteOn event (for jack\_nframes\_t=uint32\_t, size\_t=uint64\_t):

	+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
	|   time    |         size          |  MIDI  |
	+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
	|00 00 00 00|00 00 00 00 00 00 00 03|09 4a 7f|
	+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
	|           |    3 byte payload     |        |

## Jack OSC
The Jack MIDI implementation is indifferent to the buffer data being routed in a given event. We thus exploit this 'feature' to route standard OSC messages instead of raw MIDI bytes. When routing OSC, the raw OSC message is embedded in the buffer instead. For interoperability with existing OSC libraries and [NetJack](http://jackaudio.org/netjack), messages are always encoded in network byte-order.

Example OSC '/hello ,s world' message (for jack\_nframes\_t=uint32\_t, size\_t=uint64\_t):

	+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
	|   time    |         size          |            network byte-encoded OSC message               |
	+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
	|00 00 00 00|00 00 00 00 00 00 00 14|2f 68 65 6c 6c 6f 00 00 2c 73 00 00 77 6f 72 6c 64 00 00 00|
	+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
	|           |   20 byte payload     | /  h  e  l  l  o        ,  s        w  o  r  l  d         |

It makes not much sense (and calls for a lot of trouble) to route OSC bundles via Jack as the sole purpose of bundles - to associate a timestamp to one or several messages - is futile in the Jack graph as each Jack OSC event has its own sample accurate time stamp already.

**We therefore only inject and handle raw network byte-encoded OSC messages in JACK, OSC bundles are NOT allowed.**

## Header *jack\_osc.h*
Instead of all the 'jack\_midi\_\*' types and functions, we propose to use equivalent macros 'jack\_osc\_\*' to make it clear that we actually handle OSC messages via Jack MIDI ports. We provide a ready-to-use *jack\_osc.h* header for developers to include into their projects. It defines the following macros:

	#define JACK_DEFAULT_OSC_TYPE           JACK_DEFAULT_MIDI_TYPE
	#define JACK_EVENT_TYPE__OSC            "OSC"

	typedef jack_midi_data_t                jack_osc_data_t;
	typedef jack_midi_event_t               jack_osc_event_t;

	#define jack_osc_get_event_count        jack_midi_get_event_count
	#define jack_osc_event_get              jack_midi_event_get
	#define jack_osc_clear_buffer           jack_midi_clear_buffer
	#define jack_osc_max_event_size         jack_midi_max_event_size
	#define jack_osc_event_reserve          jack_midi_event_reserve
	#define jack_osc_event_write            jack_midi_event_write
	#define jack_osc_get_lost_event_count   jack_midi_get_lost_event_count

**It is not necessary to use this header since it contains transparent macros referring to the Jack MIDI API, but developers are encouraged to use this header (or at least its definitions) to make their code more clear.**

## MIDI - OSC clash
So we are routing OSC messages via Jack MIDI ports, doesn't this interfere with Jack MIDI routing in the first place?

No, it does not.

The user ideally should only connect event ports of the same type (see next section), e.g. MIDI-MIDI or OSC-OSC. But even if the user makes an inappropriate connection (MIDI-OSC or OSC-MIDI), there is no clash of events:

* OSC messages have a '/' (2f) as their first byte which is no valid MIDI status byte. MIDI clients thus should gracefully ignore OSC messages.

* Jack MIDI data in turn never has a '/' (2f) as its first byte and should thus gracefully be ignored by OSC clients.

## Metadata
To signal the user, Jack clients and patch bays that a given Jack MIDI port is used for OSC routing, we use the Jack [metadata API](http://jackaudio.org/metadata) to set the event type of the port to OSC. _Note: the metadata API currently is only implemented for Jack1, not yet for Jack2_.

The corresponding key for the event type is defined by the [Jackey](https://github.com/drobilla/jackey.git) header library. It is considered good programming practice to include this header if you deal with alternative signal (CV) and event ports (OSC) in Jack. It defines the metadata key "http://jackaudio.org/metadata/event-types" which should be set to "OSC" when used for OSC routing. By doing so, other Jack clients and patch bays can query for the port event type and act accordingly, e.g. patch bays may color the OSC ports differently and prevent the user from connecting them to MIDI ports.

After registering a Jack MIDI port you want to route OSC over, you therefore must set the corresponding metadata like this:

	#include <jack/metadata.h>

	jack_uuid_t uuid = jack_port_uuid(port);
	jack_set_property(client, uid, "http://jackaudio.org/metadata/event-types", "OSC", NULL);

or even better, using definitions in *jackey.h* and *jack\_osc.h* ...

	#include <jack/metadata.h>
	#include <jackey.h>
	#include <jack_osc.h>
	
	jack_uuid_t uuid = jack_port_uuid(port);
	jack_set_property(client, uuid, JACKEY_EVENT_TYPES, JACK_EVENT_TYPE__OSC, NULL);

Do not forget to clean up the metadata database before unregistering an OSC port.

	jack_remove_property(client, uuid, "http://jackaudio.org/metadata/event-types");

or even better using definitions in *jackey.h*...

	jack_remove_property(client, uuid, JACKEY_EVENT_TYPES);

To check whether a given Jack event port supports OSC messages you can do.

	#include <string.h>
	#include <stdio.h>

	char *value = NULL;
	char *type = NULL;
	if( (jack_get_property(uuid, JACKEY_EVENT_TYPES, &value, &type) == 0) &&
			(strstr(value, JACK_EVENT_TYPE__OSC) != NULL) )
		printf("This port routes OSC!\n");
	jack_free(value);
	jack_free(type);

## Realtime safe OSC libraries
If you want to parse, create and manipulate OSC messages in the Jack process thread, you'll need a realtime safe OSC library.

* [rtosc](https://github.com/fundamental/rtosc)

## Jack patch bays with support for OSC routing

* [Patchage](http://drobilla.net/software/patchage/) (SVN version only)

## Jack clients with support for OSC routing

* [Tjost](https://github.com/OpenMusicKontrollers/Tjost)

## Minimal working example
This is a *minimal\_example.c* for a simple Jack OSC filter. It registers two OSC ports, passes thru all OSC messages with a '\hello' path, stops the program upon an '/exit' message or emits an '/invalid' message otherwise.

	#include <stdio.h>
	#include <string.h>
	
	#include <jack/jack.h>
	#include <jack/metadata.h>
	
	#include <jackey.h>
	#include <jack_osc.h>
	
	static jack_port_t *osc_in = NULL;
	static jack_port_t *osc_out = NULL;
	static volatile int done = 0;
	
	static int
	_process(jack_nframes_t nframes, void *arg)
	{
		void *osc_in_buf = jack_port_get_buffer(osc_in, nframes);
		void *osc_out_buf = jack_port_get_buffer(osc_out, nframes);
		
		jack_osc_clear_buffer(osc_out_buf);
	
		int i;
		for(i=0; i<jack_osc_get_event_count(osc_in_buf); i++)
		{
			jack_osc_event_t jev;
			jack_osc_event_get(&jev, osc_in_buf, i);
	
			const char *path = jev.buffer;
	
			if(path[0] == '/') // check whether this is an OSC message
			{
				if(!strcmp(path, "/hello")) // only pass-thru OSC messages with '/hello' path
				{
					if(jack_osc_max_event_size(osc_out_buf) >= jev.size)
						jack_osc_event_write(osc_out_buf, jev.time, jev.buffer, jev.size);
				}
	
				else if(!strcmp(path, "/exit")) // exit program when '/exit' path is received
					done = 1;
	
				else // neither '/hello' nor '/exit'
				{
					if(jack_osc_max_event_size(osc_out_buf) >= 16)
					{
						jack_osc_data_t *buf = jack_osc_event_reserve(osc_out_buf, jev.time, 16);
						strncpy(buf, "/invalid", 12);
						strncpy(buf+12, ",", 4);
					}
				}
	
			}
			else // this maybe a wrongly routed MIDI message
				; // ignore it
		}
		return 0;
	}
	
	int
	main(int argc, char **argv)
	{
		jack_client_t *client;
		jack_uuid_t uuid_in;
		jack_uuid_t uuid_out;
	
		client = jack_client_open("Test", JackNullOption, NULL);
		jack_set_process_callback(client, _process, NULL);
	
		// register input port
		osc_in = jack_port_register(client, "osc.in", JACK_DEFAULT_OSC_TYPE, JackPortIsInput, 0);
		uuid_in = jack_port_uuid(osc_in);
		// set port event type to OSC
		jack_set_property(client, uuid_in, JACKEY_EVENT_TYPES, JACK_EVENT_TYPE__OSC, NULL);
	
		// register output port
		osc_out = jack_port_register(client, "osc.out", JACK_DEFAULT_OSC_TYPE, JackPortIsOutput, 0);
		uuid_out = jack_port_uuid(osc_out);
		// set port event type to OSC
		jack_set_property(client, uuid_out, JACKEY_EVENT_TYPES, JACK_EVENT_TYPE__OSC, NULL);
	
		jack_activate(client);
	
		while(!done)
			usleep(1000);
	
		// clean up metadata database
		jack_remove_property(client, uuid_out, JACKEY_EVENT_TYPES);
		jack_port_unregister(client, osc_out);
	
		// clean up metadata database
		jack_remove_property(client, uuid_in, JACKEY_EVENT_TYPES);
		jack_port_unregister(client, osc_in);
	
		jack_deactivate(client);
		jack_client_close(client);
	
		return 0;
	}

Compile it with:

	gcc -o minimal_example minimal_example.c `pkg-config --libs --cflags jack`
