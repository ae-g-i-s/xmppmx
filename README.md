xmppmx
======

A XMPP-to-XMPP transport written in Erlang.


xmppmx is a XMPP Transport from XMPP to XMPP. In other words, it allows you to sign into one Jabber account from another one (for instance, you can sign into your Facebook Chat account from your regular account, because Facebook Chat is simply XMPP). It utilises exmpp, an Erlang XMPP library created by the same people who produce the excellent ejabberd, ProcessOne.


WHY
======
There are mostly two reasons for this transport:
1. Spectrum, which is an otherwise perfectly fine transport written in Python (and also supports a lot more protocols than XMPP - look at http://spectrum.im), doesn't play well with UTF-8 characters in account names.
2. The previous reason alone would not really make an entirely new project necessary, but I wanted to learn Erlang. It seems like the perfect language for the job.


STATUS
======
Uh...writing the design docs :)