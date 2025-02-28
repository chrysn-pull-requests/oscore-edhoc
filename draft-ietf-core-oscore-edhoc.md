---
title: "Combining EDHOC and OSCORE"
abbrev: "EDHOC + OSCORE"
docname: draft-ietf-core-oscore-edhoc-latest
cat: std

ipr: trust200902
area: Internet
workgroup: CoRE Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: F. Palombini
    name: Francesca Palombini
    organization: Ericsson
    email: francesca.palombini@ericsson.com
 -
    ins: M. Tiloca
    name: Marco Tiloca
    org: RISE AB
    street: Isafjordsgatan 22
    city: Kista
    code: SE-16440 Stockholm
    country: Sweden
    email: marco.tiloca@ri.se
 -
    ins: R. Hoeglund
    name: Rikard Hoeglund
    org: RISE AB
    street: Isafjordsgatan 22
    city: Kista
    code: SE-16440 Stockholm
    country: Sweden
    email: rikard.hoglund@ri.se
 -
    ins: S. Hristozov
    name: Stefan Hristozov
    organization: Fraunhofer AISEC
    email: stefan.hristozov@aisec.fraunhofer.de
 -
    ins: G. Selander
    name: Goeran Selander
    organization: Ericsson
    email: goran.selander@ericsson.com

normative:
  RFC2119:
  RFC7252:
  RFC8174:
  RFC8613:
  RFC8742:
  RFC8949:
  I-D.ietf-lake-edhoc:

informative:



--- abstract

This document defines an optimization approach for combining the lightweight authenticated key exchange protocol EDHOC run over CoAP with the first subsequent OSCORE transaction. This combination reduces the number of round trips required to set up an OSCORE Security Context and to complete an OSCORE transaction using that Security Context.

--- middle

# Introduction

This document defines an optimization approach to combine the lightweight authenticated key exchange protocol EDHOC {{I-D.ietf-lake-edhoc}}, when running over CoAP {{RFC7252}}, with the first subsequent OSCORE {{RFC8613}} transaction.

This allows for a minimum number of round trips necessary to setup the OSCORE Security Context and complete an OSCORE transaction, for example when an IoT device gets configured in a network for the first time.

This optimization is desirable, since the number of protocol round trips impacts the minimum number of flights, which in turn can have a substantial impact on the latency of conveying the first OSCORE request, when using certain radio technologies.

Without this optimization, it is not possible, not even in theory, to achieve the minimum number of flights. This optimization makes it possible also in practice, since the last message of the EDHOC protocol can be made relatively small (see Section 1 of {{I-D.ietf-lake-edhoc}}), thus allowing additional OSCORE protected CoAP data within target MTU sizes.

## Terminology

{::boilerplate bcp14}

The reader is expected to be familiar with terms and concepts defined in CoAP {{RFC7252}}, CBOR {{RFC8949}}, CBOR sequences {{RFC8742}}, OSCORE {{RFC8613}} and EDHOC {{I-D.ietf-lake-edhoc}}.

# Background

EDHOC is a 3-message key exchange protocol. Section 7.2 of {{I-D.ietf-lake-edhoc}} specifies how to transport EDHOC over CoAP: the EDHOC data (referred to as "EDHOC messages") are transported in the payload of CoAP requests and responses.

This draft deals with the case of the Initiator acting as CoAP Client and the Responder acting as CoAP Server; instead, the case of the Initiator acting as CoAP Server cannot be optimized by using this approach.

That is, the CoAP Client sends a POST request containing EDHOC message_1 to a reserved resource at the CoAP Server. This triggers the EDHOC exchange on the CoAP Server, which replies with a 2.04 (Changed) Response containing EDHOC message_2. Finally, the CoAP Client sends EDHOC message_3, as a CoAP POST request to the same resource used for EDHOC message_1. The Content-Format of these CoAP messages may be set to "application/edhoc".

After this exchange takes place, and after successful verifications specified in the EDHOC protocol, the Client and Server derive the OSCORE Security Context, as specified in Section 7.2.1 of {{I-D.ietf-lake-edhoc}}. Then, they are ready to use OSCORE.

This sequential way of running EDHOC and then OSCORE is specified in {{fig-non-combined}}. As shown in the figure, this mechanism takes 3 round trips to complete.

~~~~~~~~~~~~~~~~~
   CoAP Client                                  CoAP Server
        | ------------- EDHOC message_1 ------------> |
        |                                             |
        | <------------ EDHOC message_2 ------------- |
        |                                             |
EDHOC verification                                    |
        |                                             |
        | ------------- EDHOC message_3 ------------> |
        |                                             |
        |                                    EDHOC verification
        |                                             +
OSCORE Sec Ctx                                OSCORE Sec Ctx
  Derivation                                     Derivation
        |                                             |
        | ------------- OSCORE Request -------------> |
        |                                             |
        | <------------ OSCORE Response ------------- |
        |                                             |
~~~~~~~~~~~~~~~~~
{: #fig-non-combined title="EDHOC and OSCORE run sequentially" artwork-align="center"}

The number of roundtrips can be minimized as follows. Already after receiving EDHOC message_2 and before sending EDHOC message_3, the CoAP Client has all the information needed to derive the OSCORE Security Context.

This means that the Client can potentially send at the same time both EDHOC message_3 and the subsequent OSCORE Request. On a semantic level, this approach practically requires to send two separate REST requests at the same time.

The high level message flow of running EDHOC and OSCORE combined is shown in {{fig-combined}}.

Defining the specific details of how to transport the data and of their processing order is the goal of this specification, as defined in {{edhoc-in-oscore}}.

~~~~~~~~~~~~~~~~~
   CoAP Client                                  CoAP Server
        | ------------- EDHOC message_1 ------------> |
        |                                             |
        | <------------ EDHOC message_2 ------------- |
        |                                             |
EDHOC verification                                    |
        +                                             |
  OSCORE Sec Ctx                                      |
    Derivation                                        |
        |                                             |
        | ---- EDHOC message_3 + OSCORE Request ----> |
        |                                             |
        |                                    EDHOC verification
        |                                             +
        |                                     OSCORE Sec Ctx
        |                                        Derivation
        |                                             |
        | <------------ OSCORE Response ------------- |
        |                                             |
~~~~~~~~~~~~~~~~~
{: #fig-combined title="EDHOC and OSCORE combined" artwork-align="center"}


# EDHOC Option {#signalling}

This section defines the EDHOC Option, used in a CoAP request to signal that the request combines EDHOC message_3 and OSCORE protected data.

The EDHOC Option has the properties summarized in {{fig-edhoc-option}}, which extends Table 4 of {{RFC7252}}. The option is Critical, Safe-to-Forward, and part of the Cache-Key. The option MUST occur at most once and is always empty. If any value is sent, the value is simply ignored. The option is intended only for CoAP requests and is of Class U for OSCORE {{RFC8613}}.

~~~~~~~~~~~
+-------+---+---+---+---+-------+--------+--------+---------+
| No.   | C | U | N | R | Name  | Format | Length | Default |
+-------+---+---+---+---+-------+--------+--------+---------+
| TBD13 | x |   |   |   | EDHOC | Empty  |   0    | (none)  |
+-------+---+---+---+---+-------+--------+--------+---------+
       C=Critical, U=Unsafe, N=NoCacheKey, R=Repeatable
~~~~~~~~~~~
{: #fig-edhoc-option title="The EDHOC Option." artwork-align="center"}

The presence of this option means that the message payload contains also EDHOC data, that must be extracted and processed as defined in {{server-processing}}, before the rest of the message can be processed.

{{fig-edhoc-opt}} shows the format of a CoAP message containing both the EDHOC data and the OSCORE ciphertext, using the newly defined EDHOC option for signalling.

~~~~~~~~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Ver| T |  TKL  |      Code     |          Message ID           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Token (if any, TKL bytes) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  OSCORE option  |   EDHOC option  | other options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-opt title="CoAP message for EDHOC and OSCORE combined - signalled with the EDHOC Option" artwork-align="center"}

# EDHOC Combined with OSCORE {#edhoc-in-oscore}

The approach defined in this specification consists of sending EDHOC message_3 inside an OSCORE protected CoAP message.

The resulting EDHOC + OSCORE request is in practice the OSCORE Request from {{fig-non-combined}}, sent to a protected resource and with the correct CoAP method and options, with the addition that it also transports EDHOC message_3.

Since EDHOC message_3 may be too large to be included in a CoAP Option, e.g. if containing a large public key certificate chain, it has to be transported through the CoAP payload.

The use of this approach is explicitly signalled by including an EDHOC Option (see {{signalling}}) in the EDHOC + OSCORE request.

## Client Processing {#client-processing}

The Client prepares an EDHOC + OSCORE request as follows.

1. Compose EDHOC message_3 as per Section 5.4.2 of {{I-D.ietf-lake-edhoc}}.

   Since the Client is the EDHOC Initiator and the used Correlation Method is 1 (see Section 3.2.4 of {{I-D.ietf-lake-edhoc}}), the EDHOC message_3 always includes the Connection Identifier C_R and CIPHERTEXT_3. Note that C_R is the OSCORE Sender ID of the Client, encoded as a bstr_identifier (see Section 5.1 of {{I-D.ietf-lake-edhoc}}).

2. Encrypt the original CoAP request as per Section 8.1 of {{RFC8613}}, using the new OSCORE Security Context established after receiving EDHOC message_2.

   Note that the OSCORE ciphertext is not computed over EDHOC message_3, which is not protected by OSCORE. That is, the result of this step is the OSCORE Request as in {{fig-non-combined}}.

3. Build a CBOR sequence {{RFC8742}} composed of two CBOR byte strings in the following order.

   * The first CBOR byte string is the CIPHERTEXT_3 of the EDHOC message_3 resulting from step 3.
   
   * The second CBOR byte string has as value the OSCORE ciphertext of the OSCORE protected CoAP request resulting from step 2.

4. Compose the EDHOC + OSCORE request, as the OSCORE protected CoAP request resulting from step 2, where the payload is replaced with the CBOR sequence built at step 3.

5. Signal the usage of this approach within the EDHOC + OSCORE request, by including the new EDHOC Option defined in {{signalling}}.

## Server Processing {#server-processing}

When receiving an EDHOC + OSCORE request, the Server performs the following steps.

1. Check the presence of the EDHOC option defined in {{signalling}}, to determine that the received request is an EDHOC + OSCORE request. If this is the case, the Server continues with the steps defined below.

2. Extract CIPHERTEXT_3 from the payload of the EDHOC + OSCORE request, as the first CBOR byte string in the CBOR sequence.

3. Rebuild EDHOC message_3, as a CBOR sequence composed of two CBOR byte strings in the following order.

   * The first CBOR byte string is the 'kid' of the Client indicated in the OSCORE option of the EDHOC + OSCORE request, encoded as a bstr_identifier (see Section 5.1 of {{I-D.ietf-lake-edhoc}}).

   * The second CBOR byte string is the CIPHERTEXT_3 retrieved at step 2.

4. Perform the EDHOC processing on the EDHOC message_3 rebuilt at step 3, including verifications, and the OSCORE Security Context derivation, as per Section 5.4.3 and Section 7.2.1 of {{I-D.ietf-lake-edhoc}}, respectively.

5. Extract the OSCORE ciphertext from the payload of the EDHOC + OSCORE request, as the value of the second CBOR byte string in the CBOR sequence.

6. Rebuild the OSCORE protected CoAP request as the EDHOC + OSCORE request, where the payload is replaced with the OSCORE ciphertext resulting from step 5.

7. Decrypt and verify the OSCORE protected CoAP request resulting from step 6, as per Section 8.2 of {{RFC8613}}, by using the new OSCORE Security Context established at step 4.

8. Process the CoAP request resulting from step 7.

If steps 4 (EDHOC processing) and 7 (OSCORE processing) are both successfully completed, the Server MUST reply with an OSCORE protected response, in order for the Client to achieve key confirmation (see Section 5.4.2 of {{I-D.ietf-lake-edhoc}}). The usage of EDHOC message_4 as defined in Section 7.1 of {{I-D.ietf-lake-edhoc}} is not applicable to the approach defined in this specification.

If step 4 (EDHOC processing) fails, the server discontinues the protocol as per Section 5.4.3 of {{I-D.ietf-lake-edhoc}} and sends an EDHOC error message, formatted as defined in Section 6.1 of {{I-D.ietf-lake-edhoc}}. In particular, the CoAP response conveying the EDHOC error message:

* MUST have Content-Format set to application/edhoc defined in Section 9.5 of {{I-D.ietf-lake-edhoc}}.

* MUST specify a CoAP error response code, i.e. 4.00 (Bad Request) in case of client error (e.g. due to a malformed EDHOC message_3), or 5.00 (Internal Server Error) in case of server error (e.g. due to failure in deriving EDHOC key material).

If step 4 (EDHOC processing) is successfully completed but step 7 (OSCORE processing) fails, the same OSCORE error handling applies as defined in Section 8.2 of {{RFC8613}}.

# Example of EDHOC + OSCORE Request # {#example}

An example based on the OSCORE test vector from Appendix C.4 of {{RFC8613}} and the EDHOC test vector from Appendix B.2 of {{I-D.ietf-lake-edhoc}} is given in {{fig-edhoc-opt-2}}. In particular, the example assumes that:

* The used OSCORE Partial IV is 0, consistently with the first request protected with the new OSCORE Security Context.

* The OSCORE Sender ID of the Client is 0x20. This corresponds to the EDHOC Connection Identifier C_R, which is encoded as the bstr_identifier 0x08 in EDHOC message_3.

* The EDHOC option is registered with CoAP option number 13.

~~~~~~~~~~~~~~~~~
   o  OSCORE option value: 0x090020 (3 bytes)

   o  EDHOC option value: - (0 bytes)

   o  C_R: 0x20 (1 byte)
   
   o  CIPHERTEXT_3: 0x5253c3991999a5ffb86921e99b607c067770e0
      (19 bytes)
   
   o  EDHOC message_3: 0x08 5253c3991999a5ffb86921e99b607c067770e0
      (20 bytes)

   o  OSCORE ciphertext: 0x612f1092f1776f1c1668b3825e (13 bytes)
      
   From there:

   o  Protected CoAP request (OSCORE message):
   
      0x44025d1f               ; CoAP 4-byte header
        00003974               ; Token
        39 6c6f63616c686f7374  ; Uri-Host Option: "localhost"
        63 090020              ; OSCORE Option
        40                     ; EDHOC Option
        ff 5253c3991999a5ffb86921e99b607c067770e0
           4d612f1092f1776f1c1668b3825e
      (57 bytes)
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-opt-2 title="Example of CoAP message with EDHOC and OSCORE combined" artwork-align="center"}

# Security Considerations

The same security considerations from OSCORE {{RFC8613}} and EDHOC {{I-D.ietf-lake-edhoc}} hold for this document.

TODO (more considerations)

# IANA Considerations

This document has the following actions for IANA.

## CoAP Option Numbers Registry ## {#iana-coap-options}

IANA is asked to enter the following option numbers to the "CoAP Option Numbers" registry defined in {{RFC7252}} within the "CoRE Parameters" registry.

\[

The CoAP option numbers 13 and 21 are both consistent with the properties of the EDHOC Option defined in {{signalling}}, and they both allow the EDHOC Option to always result in an overall size of 1 byte. This is because:

* The EDHOC option is always empty, i.e. with zero-length value; and

* Since the OSCORE option with option number 9 is always present in the CoAP request, the EDHOC option would be encoded with a maximum delta of 4 or 12, depending on its option number being 13 or 21.

At the time of writing, the CoAP option numbers 13 and 21 are both unassigned in the "CoAP Option Numbers" registry, as first available and consistent option numbers for the EDHOC option.

\]

~~~~~~~~~~~
+--------+-------+-------------------+
| Number | Name  |     Reference     |
+--------+-------+-------------------+
| TBD13  | EDHOC | [[this document]] |
+--------+-------+-------------------+
~~~~~~~~~~~
{: artwork-align="center"}

--- back

# Acknowledgments
{:numbered="false"}

The authors sincerely thank Christian Amsuess, Klaus Hartke, Jim Schaad and Malisa Vucinic for their feedback and comments in the discussion leading up to this draft.

The work on this document has been partly supported by VINNOVA and the Celtic-Next project CRITISEC; and by the H2020 project SIFIS-Home (Grant agreement 952652).
