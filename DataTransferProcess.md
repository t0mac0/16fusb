# Data Transfer Process #

## Control Transfers: ##

The transfer process starts when the PIC receives the first positive going pulse of the Sync Pattern (coming from D+) on the external interrupt pin (INT/RB0) initializing the interrupt service (ISR). At the end of the Sync Pattern, ISR receives and immediately saves the bits sent by host in a buffer (RX\_BUFFER)  until the end of packet (EOP) is detected. Each bit is read in the receiving loop at the middle of the sample.

After the EOP detection, the type of packet that has just arrived is checked. If it's a token, its destination address is checked to guarantee if data is actually for the device. These two checks are done even before removing the NRZI encoding, because the device is subjected to a maximum time to send a response to the host. This maximum time, in our case, would be easily exceeded if we wait the completion of the decoding process. If the address doesn’t match, then the next incoming data packet is discarded.

Using the first pulse of Sync Pattern as reference, which is always a positive going pulse on the pin INT/RB0, is what makes possible these early findings (type and address of the packet), since the values ​​of PIDs are fixed (obviously) and in the case of tokens, the first of the seven address bits immediately follows the last bit of the PID. Thus, we can make comparisons of data directly into NRZI in PID and ADDR fields on token packets, and in the PID field on data packet. Since the host can send new packets at anytime, the external interrupt of the PIC should always be ready to respond even if there is some internal processing in progress. When a processing is going on, the response to host shall be a NAK, indicating that the device is busy. Thus, host will send the packet again at a later time.

When the packet is a data packet, ISR copies RX\_BUFFER to RXINPUT\_BUFFER and ACTION\_FLAG is filled with a value to inform MainLoop that there's data in RXINPUT\_BUFFER to be decoded. Later, the decoding routine, on MainLoop, decodes (NRZI and bit stuffing removal) RXINPUT\_BUFFER to RXDATA\_BUFFER. So, RXDATA\_BUFFER contains the decoded data. After decoding, ACTION\_FLAG tells the MainLoop what to process depending on the type of token that data packet follows. For vendor/class requests MainLoop transfer control to custom code via VendorRequest or ProcessOut calling. MainLoop is also responsible for build answers for all standard requests.

If data available in RXDATA\_BUFFER are from a SETUP token, the MainLoop (or custom code) must build the right response in TX\_BUFFER if the request has been device-to-host, or perform some other operation on that basis if the request had been host-to-device. For OUT token data, processing is similar to host-to-device request. If it was a device-to-host request , then the host sends an IN token. At this point a new interrupt starts, but this time after receiving the token, the ISR enters the sending loop of TX\_BUFFER.

## Interrupt Transfers: ##

The process is the same used in control transfers except that on interrupt transfers there are no setup stages, so no SETUP packets. IN requests are sent without any Setup before and answer for them must be on INT\_TX\_BUFFER. The answer for interrupt transfer is often preprared by some code running in loop (Main label in main.asm), ever called by MainLoop.

Out packets are also sent to ProcessOut, but ACTION\_FLAG bit 2 will be setted for interrupt transfers. The data are avaliable in RXDATA\_BUFFER as in control transfer.