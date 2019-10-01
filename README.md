# Airbnb MaxScale

Airbnb MaxScale is a fork of MariaDB's MaxScale 1.3. The motivation for this project is to use a database proxy with connection pooling to significantly reduce the number of direct connections to our MySQL databases. Airbnb MaxScale had been deployed in production since early 2016, and it is currently powering all core MySQL databases used by the Airbnb web application.

See the original MariaDB MaxScale [README](README_MARIADB.md) for more information about MaxScale.

## Airbnb Features

We have implemented connection pooling and other interesting features.

### Connection Pooling

MaxScale is an intelligent router between applications and MySQL servers. In MariaDB MaxScale, the connection management model is a one-to-one connection between a client connection and backend connection. In Airbnb Maxscale, connection pooling is implemented by multiplexing N client connections over a fixed number of M connections to a backend MySQL server, where N can be significantly larger than M. After a client connection request completes successful authentication with a backend MySQL server, Airbnb MaxScale severs the link between the backend connection and client connection and parks it in a connection pool of the backend server. The server connection pool size is configurable, and is typically a small number. When receiving a query on a client connection, MaxScale picks a backend connection in the pool, links it with the client connection, and forwards the query to the backend MySQL server. MaxScale understands the transaction context of a client session and therefore it knows to keep the linked backend connection until transaction commits. The link must be kept and used for forwarding the query result back to the client.

### Requests Throttling

The server connection pool size is usually configured to 10 in our production environment. Running many instances of Airbnb MaxScale proxy servers, there are typically several hundred database connections to a MySQL server. Previously, only a small portion of connections would be in active use. When an underlying storage outage happens or a bad, expensive query hits the database, query execution becomes slow and the server connection pool runs out on each MaxScale proxy server instance. We take this symptom as a signal that the backend MySQL server may have a spike of concurrent threads running, and MaxScale proactively throttles client requests by killing client connections. In production, the request throttling had proven to be very useful in preventing database incidents due to transient storage system outages.

### Denylist Query Rejection

MaxScale uses an embedded MySQL parser for query routing. It builds a parse tree for every MySQL query in its query classifier module. Airbnb MaxScale leverages the query parse tree for bad query blacklisting. The motivation to build this feature came from a Ruby VM heap memory corruption incident in our production environment. Because of the memory corruption, Rails' ActiveRecord generated a MySQL query statement which was corrupted in such way that its conditional predicate was effectively removed. For instance, one corrupted query that we were lucky to catch before it caused permanent damage in production was a delete statement with `where 0 = 0`. This had put our production databases in danger of serious corruption. This [blog post](http://webuild.envato.com/blog/tracking-down-ruby-heap-corruption/) has a detailed explanation of the nasty Ruby heap corruption problem. Airbnb MaxScale looks for existence of malformed equality conditions in update and delete statements by inspecting the predicate in the query parse tree and rejecting them.

### Monitoring

Airbnb MaxScale implements minutely internal metrics collection and exposes them for real time monitoring. A new `stats` command is added in MaxAdmin tool. External stats agent can be created to pull stats from a running MaxScale server periodically.

## Build

Airbnb MaxScale was forked off the MaxScale 1.3 development branch. All Airbnb features are in the branch `connection_proxy`.

Before building Airbnb MaxScale, make sure you have all prerequisites installed on the build machine. For the sake of example, the Airbnb Maxscale source is in `/srv/dbproxy/MaxScale`, and the dependency libraries are installed in `/srv/dbproxy`.

The following builds a Debian release package. The release package name has an AIRBNB version number in it.

```
mkdir release_maxscale
cd release_maxscale
cmake /srv/dbproxy/MaxScale -DCMAKE_BUILD_TYPE=RelWithDebugInfo -DMYSQL_DIR=/srv/dbproxy/mariadb-10.0.20-linux-x86_64/include/mysql -DEMBEDDED_LIB=/srv/dbproxy/mariadb-10.0.20-linux-x86_64/lib/libmysqld.so -DMYSQLCLIENT_LIBRARIES=/srv/dbproxy/mariadb-10.0.20-linux-x86_64/lib/libmysqlclient.so -DERRMSG=/srv/dbproxy/mariadb-10.0.20-linux-x86_64/share/english/errmsg.sys -DBUILD_TESTS=Y -DWITH_SCRIPTS=Y -DPACKAGE=Y -DCMAKE_INSTALL_PREFIX=/srv/dbproxy
make
make package
```

For a debug build it is slightly different:

```
mkdir debug_maxscale
cd debug_maxscale
cmake /srv/dbproxy/src -DCMAKE_BUILD_TYPE=Debug -DMYSQL_DIR=/srv/dbproxy/mariadb-10.0.20-linux-x86_64/include/mysql -DEMBEDDED_LIB=/srv/dbproxy/mariadb-10.0.20-linux-x86_64/lib/libmysqld.so -DMYSQLCLIENT_LIBRARIES=/srv/dbproxy/mariadb-10.0.20-linux-x86_64/lib/libmysqlclient.so -DERRMSG=/srv/dbproxy/mariadb-10.0.20-linux-x86_64/share/english/errmsg.sys -DBUILD_TESTS=Y -DWITH_SCRIPTS=N -DCMAKE_INSTALL_PREFIX=$HOME/debug_maxscale
make
make install
```

For more information about Airbnb MaxScale, please read [documentations](Documentation/Airbnb).

We welcome feature ideas and pull requests for bugs and features. We are also happy to work with other parties to merge this branch back into the upstream branch for the benefit of the broader community


Apache License
                           Version 2.0, January 2004
                        https://www.apache.org/licenses/

   TERMS AND CONDITIONS FOR USE, REPRODUCTION, AND DISTRIBUTION

   1. Definitions.

      "License" shall mean the terms and conditions for use, reproduction,
      and distribution as defined by Sections 1 through 9 of this document.

      "Licensor" shall mean the copyright owner or entity authorized by
      the copyright owner that is granting the License.

      "Legal Entity" shall mean the union of the acting entity and all
      other entities that control, are controlled by, or are under common
      control with that entity. For the purposes of this definition,
      "control" means (i) the power, direct or indirect, to cause the
      direction or management of such entity, whether by contract or
      otherwise, or (ii) ownership of fifty percent (50%) or more of the
      outstanding shares, or (iii) beneficial ownership of such entity.

      "You" (or "Your") shall mean an individual or Legal Entity
      exercising permissions granted by this License.

      "Source" form shall mean the preferred form for making modifications,
      including but not limited to software source code, documentation
      source, and configuration files.

      "Object" form shall mean any form resulting from mechanical
      transformation or translation of a Source form, including but
      not limited to compiled object code, generated documentation,
      and conversions to other media types.

      "Work" shall mean the work of authorship, whether in Source or
      Object form, made available under the License, as indicated by a
      copyright notice that is included in or attached to the work
      (an example is provided in the Appendix below).

      "Derivative Works" shall mean any work, whether in Source or Object
      form, that is based on (or derived from) the Work and for which the
      editorial revisions, annotations, elaborations, or other modifications
      represent, as a whole, an original work of authorship. For the purposes
      of this License, Derivative Works shall not include works that remain
      separable from, or merely link (or bind by name) to the interfaces of,
      the Work and Derivative Works thereof.

      "Contribution" shall mean any work of authorship, including
      the original version of the Work and any modifications or additions
      to that Work or Derivative Works thereof, that is intentionally
      submitted to Licensor for inclusion in the Work by the copyright owner
      or by an individual or Legal Entity authorized to submit on behalf of
      the copyright owner. For the purposes of this definition, "submitted"
      means any form of electronic, verbal, or written communication sent
      to the Licensor or its representatives, including but not limited to
      communication on electronic mailing lists, source code control systems,
      and issue tracking systems that are managed by, or on behalf of, the
      Licensor for the purpose of discussing and improving the Work, but
      excluding communication that is conspicuously marked or otherwise
      designated in writing by the copyright owner as "Not a Contribution."

      "Contributor" shall mean Licensor and any individual or Legal Entity
      on behalf of whom a Contribution has been received by Licensor and
      subsequently incorporated within the Work.

   2. Grant of Copyright License. Subject to the terms and conditions of
      this License, each Contributor hereby grants to You a perpetual,
      worldwide, non-exclusive, no-charge, royalty-free, irrevocable
      copyright license to reproduce, prepare Derivative Works of,
      publicly display, publicly perform, sublicense, and distribute the
      Work and such Derivative Works in Source or Object form.

   3. Grant of Patent License. Subject to the terms and conditions of
      this License, each Contributor hereby grants to You a perpetual,
      worldwide, non-exclusive, no-charge, royalty-free, irrevocable
      (except as stated in this section) patent license to make, have made,
      use, offer to sell, sell, import, and otherwise transfer the Work,
      where such license applies only to those patent claims licensable
      by such Contributor that are necessarily infringed by their
      Contribution(s) alone or by combination of their Contribution(s)
      with the Work to which such Contribution(s) was submitted. If You
      institute patent litigation against any entity (including a
      cross-claim or counterclaim in a lawsuit) alleging that the Work
      or a Contribution incorporated within the Work constitutes direct
      or contributory patent infringement, then any patent licenses
      granted to You under this License for that Work shall terminate
      as of the date such litigation is filed.

   4. Redistribution. You may reproduce and distribute copies of the
      Work or Derivative Works thereof in any medium, with or without
      modifications, and in Source or Object form, provided that You
      meet the following conditions:

      (a) You must give any other recipients of the Work or
          Derivative Works a copy of this License; and

      (b) You must cause any modified files to carry prominent notices
          stating that You changed the files; and

      (c) You must retain, in the Source form of any Derivative Works
          that You distribute, all copyright, patent, trademark, and
          attribution notices from the Source form of the Work,
          excluding those notices that do not pertain to any part of
          the Derivative Works; and

      (d) If the Work includes a "NOTICE" text file as part of its
          distribution, then any Derivative Works that You distribute must
          include a readable copy of the attribution notices contained
          within such NOTICE file, excluding those notices that do not
          pertain to any part of the Derivative Works, in at least one
          of the following places: within a NOTICE text file distributed
          as part of the Derivative Works; within the Source form or
          documentation, if provided along with the Derivative Works; or,
          within a display generated by the Derivative Works, if and
          wherever such third-party notices normally appear. The contents
          of the NOTICE file are for informational purposes only and
          do not modify the License. You may add Your own attribution
          notices within Derivative Works that You distribute, alongside
          or as an addendum to the NOTICE text from the Work, provided
          that such additional attribution notices cannot be construed
          as modifying the License.

      You may add Your own copyright statement to Your modifications and
      may provide additional or different license terms and conditions
      for use, reproduction, or distribution of Your modifications, or
      for any such Derivative Works as a whole, provided Your use,
      reproduction, and distribution of the Work otherwise complies with
      the conditions stated in this License.

   5. Submission of Contributions. Unless You explicitly state otherwise,
      any Contribution intentionally submitted for inclusion in the Work
      by You to the Licensor shall be under the terms and conditions of
      this License, without any additional terms or conditions.
      Notwithstanding the above, nothing herein shall supersede or modify
      the terms of any separate license agreement you may have executed
      with Licensor regarding such Contributions.

   6. Trademarks. This License does not grant permission to use the trade
      names, trademarks, service marks, or product names of the Licensor,
      except as required for reasonable and customary use in describing the
      origin of the Work and reproducing the content of the NOTICE file.

   7. Disclaimer of Warranty. Unless required by applicable law or
      agreed to in writing, Licensor provides the Work (and each
      Contributor provides its Contributions) on an "AS IS" BASIS,
      WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
      implied, including, without limitation, any warranties or conditions
      of TITLE, NON-INFRINGEMENT, MERCHANTABILITY, or FITNESS FOR A
      PARTICULAR PURPOSE. You are solely responsible for determining the
      appropriateness of using or redistributing the Work and assume any
      risks associated with Your exercise of permissions under this License.

   8. Limitation of Liability. In no event and under no legal theory,
      whether in tort (including negligence), contract, or otherwise,
      unless required by applicable law (such as deliberate and grossly
      negligent acts) or agreed to in writing, shall any Contributor be
      liable to You for damages, including any direct, indirect, special,
      incidental, or consequential damages of any character arising as a
      result of this License or out of the use or inability to use the
      Work (including but not limited to damages for loss of goodwill,
      work stoppage, computer failure or malfunction, or any and all
      other commercial damages or losses), even if such Contributor
      has been advised of the possibility of such damages.

   9. Accepting Warranty or Additional Liability. While redistributing
      the Work or Derivative Works thereof, You may choose to offer,
      and charge a fee for, acceptance of support, warranty, indemnity,
      or other liability obligations and/or rights consistent with this
      License. However, in accepting such obligations, You may act only
      on Your own behalf and on Your sole responsibility, not on behalf
      of any other Contributor, and only if You agree to indemnify,
      defend, and hold each Contributor harmless for any liability
      incurred by, or claims asserted against, such Contributor by reason
      of your accepting any such warranty or additional liability.

   END OF TERMS AND CONDITIONS

   APPENDIX: How to apply the Apache License to your work.

      To apply the Apache License to your work, attach the following
      boilerplate notice, with the fields enclosed by brackets "[]"
      replaced with your own identifying information. (Don't include
      the brackets!)  The text should be enclosed in the appropriate
      comment syntax for the file format. We also recommend that a
      file or class name and description of purpose be included on the
      same "printed page" as the copyright notice for easier
      identification within third-party archives.

   Copyright [2019] [Rolando Gopez Lacuata]

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       https://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.

