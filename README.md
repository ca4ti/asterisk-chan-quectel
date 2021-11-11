Channel driver for Quectel and Simcom modules 
=================================================
This should work with Quectel modules such as EC20, EC21, EC25, EG9x and others with voice over USB capability.

I'm not a C programmer at all and this write-up may help those interested in improving this driver and also for supporting other modules such as those from Telit, U-blox, Sierra Wireless etc. 

A key part is understanding call handling by the driver. Everything revolves around URCs (Unsolicted Result Code) received at the modem terminal when certain events occur. So for Huawei when dialling out an ^ORIG URC is generated, when an outgoing or incoming call is connected there apperas a ^CONN URC and when a call is terminated by either party a ^CEND URC is received. The RING URC for incoming calls is universal but all the others mentioned are specific to Huawei and appropriate changes are to be made in <a href="https://github.com/IchthysMaranatha/asterisk-chan-quectel/blob/master/at_response.h">at_response.h</a> and <a href="https://github.com/IchthysMaranatha/asterisk-chan-quectel/blob/master/at_response.c">at_response.c</a> to handle calls correctly. 

In many modules, when AT^DSCI=1 is set, URC for call status indication is generated. For example the URC ^DSCI: 4,1,3,0,+XXXXXXXXX,145 is received indicating that an incoming voice call has been connected. The URC ^DSCI: 3,0,6,0,XXXXXXX,129 indicates an outgoing voice call has terminated. ^DSCI can be activated on modules by many manufacturers such as Quectel, Simcom, Telit, Sierra Wireless etc. There may be some differences, for example my Simcom 7600 module does not generate the call termination URC.

Armed with this knowledge and using AT^DSCI=1 as an intialization command, we can now set up the Huawei ^ORIG URC <a href="https://github.com/IchthysMaranatha/asterisk-chan-quectel/blob/b6f6a389f2d8cc0b1ba183f6ebe7f140e03730af/at_response.h#L32">here</a> to handle entire call management (initiatiing, connecting in or out, terminating) through code block <a href="https://github.com/IchthysMaranatha/asterisk-chan-quectel/blob/b6f6a389f2d8cc0b1ba183f6ebe7f140e03730af/at_response.c#L612-L737">here</a> with necessary changes.

Once this is correctly done, we need to make sure that USB voice is properly activated for the call. For the Quectel module AT+QPCMV=1,0 enables voice over the USB emulated serial port. Unlike Huawei, this needs to be done before dialling out or receivng the call. We can also set it up as an initialisation command, but problem with this is that after a few calls voice may stop working and again a reset with AT+QPCMV=0 followed by AT+QPCMV=1,0 is needed. So I've appended AT+QPCMV=0 and AT+QPCMV=1,0 with every ATD and ATA command as done <a href="https://github.com/IchthysMaranatha/asterisk-chan-quectel/blob/b6f6a389f2d8cc0b1ba183f6ebe7f140e03730af/at_command.c#L526">here</a> and <a href="https://github.com/IchthysMaranatha/asterisk-chan-quectel/blob/b6f6a389f2d8cc0b1ba183f6ebe7f140e03730af/at_command.c#L571">here</a> Voice can also be set up over virtual USB sound card with AT+QPCMV=2,1 but I'll touch upon that later.

For Simcom voice activation over USB with AT+CPCMREG=1 can be done only after a call is connected. It will persist even after a call but with the same issue that after a few calls it will need a reset with AT+CPCMREG=0. So one can use CPCMREG=0 before every DIAL or ANSWER and CPCMREG=1 after call connection. The CPCMREG=1 cannot imeediately follow ATD OR ATA as it throw an error or freeze up (I do not know if this is a problem with just my module). Still working out the kinks with Sim7600 and will upload code in a few days.

Way forward: This <a href="http://laforge.gnumonks.org/blog/20170902-cellular_modems-voice/">justified rant</a> by a professional in the industry opines that the virtual sound card solution is the way voice should be provided. We can already see that in few modules from Quectel, Telit, u-blox, Sierra Wireless etc. In such a scenario, development needs to be done for an integrated alsa voice channel for this driver. One can also use this driver as a control channel and chan_console or chan_alsa as the voice channel. So for EC25, one could set up AT+QPCMV=1,2 and then use bridged alsa channel which points to the modem sound card for every call. For EC25, one time USB configuration enabling UAC also needs to done as per doc <a href="https://forums.quectel.com/uploads/short-url/xnztA07u1c4hilREu66LYIY6NGR.pdf">here</a>

Those with problems with chan_mobile with certain controllers such as the in built Raspberry Pi one may find a solution <a href="https://blog.maplein.com/2021/09/fix-for-audio-issues-with-asterisk-and.html">here</a>

I won't accept donations, but if you wish you could donate to the person I've forked this from. Or if so inclined, you could donate here https://bethmyriam.org/ 

Thanks be to the Father of Lights from Whom are all good things.

Building:
----------

    $ ./bootstrap
    $ ./configure --with-astversion=16.20
    $ make
    $ make install
    copy quectel.conf to /etc/asterisk

If you run a different version of Asterisk, you'll need to update the
`16.20` as appropriate, obviously.

If you did not `make install` Asterisk in the usual location and configure
cannot find the asterisk header files in `/usr/include/asterisk`, you may
optionally pass `--with-asterisk=PATH/TO/INCLUDE`.

Here is an example for the dialplan:
------------------------------------

**WARNING**: *This example uses the raw SMS message passed to System() directly.
No sane person would do that with untrusted data without escaping/removing the
single quotes.*

    [quectel-incoming]
    exten => sms,1,Verbose(Incoming SMS from ${CALLERID(num)} ${BASE64_DECODE(${SMS_BASE64})})
    exten => sms,n,System(echo '${STRFTIME(${EPOCH},,%Y-%m-%d %H:%M:%S)} - ${QUECTELNAME} - ${CALLERID(num)}: ${BASE64_DECODE(${SMS_BASE64})}' >> /var/log/asterisk/sms.txt)
    exten => sms,n,Hangup()

    exten => ussd,1,Verbose(Incoming USSD: ${BASE64_DECODE(${USSD_BASE64})})
    exten => ussd,n,System(echo '${STRFTIME(${EPOCH},,%Y-%m-%d %H:%M:%S)} - ${QUECTELNAME}: ${BASE64_DECODE(${USSD_BASE64})}' >> /var/log/asterisk/ussd.txt)
    exten => ussd,n,Hangup()

    exten => s,1,Dial(SIP/2001@othersipserver)
    exten => s,n,Hangup()

    [othersipserver-incoming]

    exten => _X.,1,Dial(Quectel/r1/${EXTEN})
    exten => _X.,n,Hangup

    exten => *#123#,1,QuectelSendUSSD(quectel0,${EXTEN})
    exten => *#123#,n,Answer()
    exten => *#123#,n,Wait(2)
    exten => *#123#,n,Playback(vm-goodbye)
    exten => *#123#,n,Hangup()

    exten => _#X.,1,QuectelSendSMS(quectel0,${EXTEN:1},"Please call me",1440,yes,"magicID")
    exten => _#X.,n,Answer()
    exten => _#X.,n,Wait(2)
    exten => _#X.,n,Playback(vm-goodbye)
    exten => _#X.,n,Hangup()

You can also use this:
----------------------

Call using a specific group:

    exten => _X.,1,Dial(Quectel/g1/${EXTEN})

Call using a specific group in round robin:

    exten => _X.,1,Dial(Quectel/r1/${EXTEN})

Call using a specific quectel:

    exten => _X.,1,Dial(Quectel/quectel0/${EXTEN})

Call using a specific provider name:

    exten => _X.,1,Dial(Quectel/p:PROVIDER NAME/${EXTEN})

Call using a specific IMEI:

    exten => _X.,1,Dial(Quectel/i:123456789012345/${EXTEN})

Call using a specific IMSI prefix:

    exten => _X.,1,Dial(Quectel/s:25099203948/${EXTEN})

How to store your own number:

    quectel cmd quectel0 AT+CPBS=\"ON\"
    quectel cmd quectel0 AT+CPBW=1,\"+123456789\",145


Other CLI commands:
-------------------

    quectel reset <device>
    quectel restart gracefully <device>
    quectel restart now <device>
    quectel restart when convenient <device>
    quectel show device <device>
    quectel show devices
    quectel show version
    quectel sms <device> number message
    quectel ussd <device> ussd
    quectel stop gracefully <device>
    quectel stop now <device>
    quectel stop when convenient <device>
    quectel start <device>
    quectel restart gracefully <device>
    quectel restart now <device>
    quectel restart when convenient <device>
    quectel remove gracefully <device>
    quectel remove now <device>
    quectel remove when convenient <device>
    quectel reload gracefully
    quectel reload now
    quectel reload when convenient

For reading installation notes please look to INSTALL file.


Gain control and Jitter buffer
--------------------------------


<img src="https://cloud.githubusercontent.com/assets/6702424/26686554/9253bc18-46ed-11e7-9bce-cad8e2396435.png" 
width="800px" height="" />


In order to perform good quality calls you will need to take care of:

* **Automatic gain control**:

chan_quectel does not control the gain of the audio stream it receive.
This result of Alice hearing Bob's voice loud and noisy.
It is possible to manually manage the gain in *quectel.conf* but
the better option is by far to apply automatic gain control with
the dialplan function AGC.


* **Jitter buffer**:

Since asterisk 12 it is no longer possible to enable Jitter buffer
in quectel.conf it has to be applied in the dialplan.
The lack of Jitter buffer result in severe loss in the transport
of the voice from Bob to Alice. 


#### Dialplan example

To set JITTERBUFFER and AGC in the dialplan on the appropriate channel
regardless of who is initiating the call we will have to use
the "b" option of Dial:

b( context^exten^priority )

Before initiating an outgoing call, Gosub to the specified
location using the newly created channel. 

The Gosub will be executed for each destination channel."

    [from-quectel]
    ; This will be executed by an indbound Quectel channel ( call initiated on the quectel side )
    exten => _[+0-9].,1,Dial(SIP/bob,b(from-quectel^outbound^1)) ;

    ; This will be executed by an outbound SIP channel ( channel generated by dial )
    exten => outbound,1,Set(JITTERBUFFER(adaptive)=default)
    same => n,Set(AGC(rx)=4000)
    same => n,Return()

    [from-sip]
    ; This will be executed by an inbound SIP channel ( call initiated on the SIP side )
    exten => _[+0-9].,1,Set(JITTERBUFFER(adaptive)=2000,1600,120)
    same => n,Set(AGC(rx)=4000)
    same => n,Dial(Quectel/i:${IMEI_OF_MY_QUECTEL}/${NUMBER_OF_BOB}) 


Note: To use automatic gain control dialplan function (AGC) you will need
to compile Asterisk with func_speex ( see in menuselect ). 
On raspberry Pi you will need to compile and install speex and speexdsp yourself,
the version of speex provided by the depos does not support AGC.
(because compiled with fixed point instead of floating point)
see: [HOWTO](https://gist.github.com/garronej/01f0dac45efe9161969a83890c019efa)


For additional information about Huawei dongle usage look to
chan\_dongle Wiki at http://wiki.e1550.mobi and chan\_quectel project home at
https://github.com/t4rd15/asterisk-chan-quectel/
