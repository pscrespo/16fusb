# Adding Functionalities #

The 16FUSB was developed as a core which, through an interface, is able to provide code that will support real functionality to the device. Without the code that add functionality, the firmware responds to all standards requests, fulfilling the entire protocol (including device enumeration), but with no practical application. Following we'll see how codes for custom requests can be added to the core.

First let's see how the source code is organized. The source files can be divided into two groups: core files and interface files.

Core Files:
It's the files that make possible all USB communication. No need to be modified to add features.

  * isr.asm: File that contains the code responsible for interrupt service routine (ISR) described above.
  * usb.asm: Contains the MainLoop code, also already described above. It makes the integration of all interface files with the firmware core.
  * func.asm: It has general functions used by core.
  * stdreq.asm: Implements the answer for all mandatory standard requests.

Interface files:
These files are the interface between the core and the user code. We shall edit them to add new functionalities.

  * main.asm: This file allows you to declare initial settings and run some task in loop. On Init label you can put anything to do after PIC reset and before it starts accepting interruptions. On Main label you can run a periodic task (eg. put in INT\_TX\_BUFFER the state of PIC pins).
  * setup.asm: All non standard requests are redirected to here via a "call" to the label VendorRequest. This is where we insert the code that handles custom control transfers (vendor/class). In short, the code flow will be on the label VendorRequest whenever we have control transfers, with data from the SETUP stage or data requested during DATA stage, ie, on device-to-host direction (IN).
  * out.asm: When receiving data from an OUT token, flow is transferred to this file at ProcessOut label. We may understand this point as a callback function for OUT packets. An OUT packet will be available here in a control transfer DATA stage of a host-to-device request as well as OUT packets coming from interrupt transfers.
  * rpt\_desc.asm: Contains the HID Report Descriptor. Change this to customize the reports.<br><br></li></ul>

## Working With Control Transfers (EP0): ##
Every control transfer starts with a Setup stage, composed by a SETUP token packet and a DATA0 data packet (see Image 5). On Table 1 we can see the Setup request format. Whenever a non standard request happens, the MainLoop calls VendorRequest. If you look at offset 0 description (Table 1), you’ll notice that VendorRequest will be called if value composed by bit 5 and 6 of bmRequestType field is not zero. These two bit defines if a request is standard, class or vendor request.<br><br>

<img src='http://imageshack.com/a/img163/3525/95rh.png' /><br><br>

<table><thead><th> <b>Offset</b> </th><th> <b>Field</b> </th><th> <b>Size</b> </th><th> <b>Value</b> </th><th> <b>Description</b> </th></thead><tbody>
<tr><td> 0             </td><td> bmRequestType </td><td> 1           </td><td> Bit-Map      </td><td> <b>D7 Data Phase Transfer Direction</b> <br> 0 = Host to Device <br> 1 = Device to Host <br> <b>D6..5 Type</b> <br> 0 = Standard <br> 1 = Class <br> 2 = Vendor <br> 3 = Reserved <br> <b>D4..0 Recipient</b> <br> 0 = Device <br> 1 = Interface <br> 2 = Endpoint <br> 3 = Other <br> 4..31 = Reserved </td></tr>
<tr><td> 1             </td><td> bRequest     </td><td> 1           </td><td> Value        </td><td> Request            </td></tr>
<tr><td> 2             </td><td> wValue       </td><td> 2           </td><td> Value        </td><td> Value              </td></tr>
<tr><td> 4             </td><td> wIndex       </td><td> 2           </td><td> Index or Offset </td><td> Index              </td></tr>
<tr><td> 6             </td><td>	wLength      </td><td> 2           </td><td> Count        </td><td> Number of bytes to transfer if there is a data phase </td></tr></tbody></table>

<br><br>
In VendorRequest routine, setup data can be read in RXDATA_BUFFER, how we can see in picture below. We can read the offsets just using the form RXDATA_BUFFER+offset, ex: RXDATA_BUFFER+2 reads wValue low. The values in message fields is part of the developer’s imagination.<br><br>

<img src='http://imageshack.com/a/img585/4272/qzit.png' /><br><br>

All kind of requests always have a transfer direction: Device-to-Host - host expects get data from device; Host-to-Device - roughly, host sends data to device.<br><br>

<h3>Device-to-Host request:</h3>

Transfer direction can be checked on bit 7 of bmRequestType field. Once we’re on device-to-host request, we need to compose the answer, because host will send IN requests to get the device response. Thus, MainLoop delegates this to VendorRequest procedure. The answer shall fill 1 to 8 offsets of TX_BUFFER (TX_BUFFER+1 … TX_BUFFER+8). We do not need fill the three others offsets, MainLoop will do it automatically for us.<br><br>

<img src='http://imageshack.com/a/img845/8130/4dk8.png' /><br><br>

Low speed devices are limited to a maximum of 8 bytes packet size. If data stage have more than 8 bytes (wLength > 8), the transaction will be divided in multiple packets. In this case you shall use the FRAME_NUMBER file register for check what part of answer host is asking.<br><br>

Example: wLength = 20 <br>
<blockquote>Host ask for first 8 bytes  → FRAME_NUMBER = 0 <br>
Host ask for second 8 bytes → FRAME_NUMBER = 1 <br>
Host ask for last 4 bytes   → FRAME_NUMBER = 2 <br>
<pre><code>DeviceToHostRequest:           <br>
    movlw   0x01<br>
    subwf   FRAME_NUMBER,W<br>
    btfsc   STATUS,Z<br>
    goto    Answer_Frame1<br>
    movlw   0x02<br>
    subwf   FRAME_NUMBER,W<br>
    btfsc   STATUS,Z<br>
    goto    Answer_Frame2<br>
<br>
Answer_Frame0:<br>
  movlw    0x55<br>
  movwf    TX_BUFFER+1<br>
<br>
  movlw    0xAA<br>
  movwf    TX_BUFFER+2<br>
<br>
  ...<br>
<br>
  movlw   0x55<br>
  movwf    TX_BUFFER+8<br>
 <br>
  return<br>
<br>
Answer_Frame1: <br>
  ...<br>
<br>
Answer_Frame2: <br>
  ...<br>
</code></pre>
<br></blockquote>

<h3>Host-to-Device request:</h3>

Non standard Host-to-Device request are also always treated firstly by VendorRequest, and only by it if we do not have data stage. For request with data stage, after Setup, host will send OUT packet and MainLoop will transfer control to ProcessOut (out.asm). At this point, RXDATA_BUFFER will reflect data stage content.<br>
<br>
For transaction with more than 8 bytes, maybe you need to know which Setup request comes OUT packets. Any Setup information will only be available in VendorRequest. This is a good chance to save some information to be used later. For example, if our requests are based in bRequest field, at VendorRequest we can save it in other register and make a query for its value later in ProcessOut to know how to proceed with the data in RXDATA_BUFFER.<br><br>

<h2>Working With Interrupt Transfers (EP1):</h2>
To handle interrupt transfer is a little different than handle control transfers. Interrupt transfers sends IN and OUT requests directly, without use a SETUP stage. Thus, when host wants to get some data from device it simple send a IN packet to EP1. If device have no data to send, a NAK is sent to host. Host may try again according to a defined timeout. When host needs to send data to device it simple send a OUT packet followed by data. As we have either SETUP nor STATUS stage in interrupt transfers, we can say that it's faster than control transfers.<br>
<br>
<h3>IN Interrupt Transfer:</h3>
The answer for IN interrupt transfers is made using the buffer INT_TX_BUFFER, the same way we do in TX_BUFFER (INT_TX_BUFFER+1 … INT_TX_BUFFER+8). After build the message the code must call PrepareIntTxBuffer and put in INT_TX_LEN the message length. This routine will adjust data toggle, calculate CRC16, insert bit stuffing and set the ACTION_FLAG bit 5 (AF_BIT_INT_TX_READY) to inform there are bytes to send in EP1 interrupt endpoint. Once the message is ready, on the next host's poll (IN packet), the ISR sends the INT_TX_BUFFER and just after clear AF_BIT_INT_TX_READY. Thus, checking the AF_BIT_INT_TX_READY device knows if the buffer was sent and if a new message can be queued on buffer.<br>
<br>
<pre><code>(main.asm)<br>
; Always get pins state and put in buffer<br>
Main:<br>
    call     GetPinsState         ; Returns in W the PORTA pins state.<br>
                    <br>
    movwf    INT_TX_BUFFER+1      ; Put pins state in buffer<br>
<br>
    movlw    0x01                 ; Send one byte.<br>
    movwf    INT_TX_LEN<br>
<br>
    call     PrepareIntTxBuffer   ; Prepare buffer<br>
<br>
    return<br>
<br>
---<br>
<br>
(main.asm)<br>
; Get pins state and queue in buffer. Do not put new pins state in buffer until host receive the queued message.<br>
Main:<br>
    btfsc    AF_BIT_INT_TX_READY  ; If there are pending message in buffer, return.<br>
    return<br>
<br>
    call     GetPinsState         ; Returns in W the PORTA pins state.<br>
                    <br>
    movwf    INT_TX_BUFFER+1      ; Put pins state in buffer<br>
<br>
    movlw    0x01                 ; Send one byte.<br>
    movwf    INT_TX_LEN<br>
<br>
    call     PrepareIntTxBuffer   ; Prepare buffer<br>
<br>
    return<br>
</code></pre>

<h3>OUT Interrupt Transfer:</h3>
Data from OUT packets on interrupt transfers arrives the ProcessOut label with data avaliable in RXDATA_BUFFER,  just like in control transfers. So, how to know whether the packet is from control or interrupt transfer? All we need to do is checking ACTION_FLAG bit 2 (AF_BIT_INTERRUPT). It's cleared for control transfers and setted for interrupt transfers.<br><br>

<h2>Working With HID:</h2>

HID specs defines use of control and interrupt transfers. IN interrupt transfer is mandatory, so one IN interrupt endpoint must be implemented. OUT interrupt endpoint is opitional. To enable HID and the EP1 IN, you must edit def.inc in the application folder. If you want to use OUT interrupt transfer with HID, just enable the EP1 OUT in the same file. Without the EP1 OUT, all HID reports sent by host to device will be via control transfer.<br>
<br>
To send message for device via control transfer, HID uses Set_Report. To get messages from device, Get_Report. Depending on the API you're using, you may invoke functions thats guarantee a specific transfer type. On Windows 7, for example, you may use HidD_GetInputReport and HidD_SetOutputReport to generate Get_Report and Set_Report, respectively. The ReadFile function always retrives a buffered message obtained by host via an IN interrupt transfer. WriteFile function send message for device using interrupt transfer if EP1 OUT is available, otherwise it use control transfer.<br>
<br>
To handle interrupt transfers with HID is same as described above. Control transfers with HID will be avaliable in VendorRequest (setup.asm) like any other class/vendor request. On host-to-device messages case (eg. Set_Report), the OUT packet will be avaliable in ProcessOut like any other OUT packet.<br>
<br>
When HID is enabled, we have only messages defined by HID specs. Thus, when a control transfer arrives VendorRequest label, we know thats is a HID message. All we have to check is the kind of HID request and/or the report ID. For Get_Report we have bRequest equals to 0x1 and for Set_Report bRequest is 0x09. To configure the Report Descriptor edit the rpt_desc.inc file. The length of the descriptor must be configured in def.inc.<br>
<br>
<br>
<br>
<h2>16FUSB RAM use:</h2>

The core of 16FUSB only uses Bank 0 including some positions of shared area (mirrored). On Bank 0, you can use free positions of shared area and LOCAL_OVERLAY section. Using LOCAL_OVERLAY you have the advantage of don’t worry about memory banks.<br>
<br>
Bank 1 and 2 are totally free and can be all used by the functionality. Don’t forget to use “banksel” directive to select the right bank for your register if you use other banks than Bank 0.<br>
<br>
Whenever you need to save information to query later, you may save it in an overlay section. Notice that in this case you cannot use LOCAL_OVERLAY as it is a temporary area, ever overwritten by core.<br>

Example of an overlay section:<br>
<pre><code>(vreq.asm)<br>
;Bank 1 file registers<br>
MYAPP_OVERLAY	UDATA_OVR 	0xA0<br>
<br>
MYREG		RES	1<br>
<br>
(out.asm)<br>
;Bank 1 file registers<br>
MYAPP_OVERLAY	UDATA_OVR 	0xA0<br>
<br>
;MYREG here will access the same content that MYREG in vreq.asm<br>
MYREG		RES	1<br>
</code></pre>

You also may simple use an udata section and just leave the linker decide the registers addresses. It still can be accessible by other objects if you use global directive. Anyway, banksel directive still must be used as you don’t know where linker will put your variable.<br>

Example:<br>
<pre><code>MYAPP_VARIABLES	UDATA<br>
<br>
MYREG		RES	1<br>
</code></pre>