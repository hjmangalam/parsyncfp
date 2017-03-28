# parsyncfp
follow-on to parsync (parallel rsync) with better startup performance.

This is the next-gen version of parsync which uses 
fpart <http://goo.gl/K1WwtD> (by <ganael.laplanche@martymac.org>) to create chunkfiles for rsync
to read, bypassing the need to wait for a complete recursive scan. 
This allows transfer of huge dir trees to start immediately, with fpart creating chunkfiles as it does the recursive descent 
with parsyncfp ingesting them and maintaining NP active rsyncs to send the files listed therein.

This is an early version which is being brought up to code parity with parsync.

It is not yet working or feature-complete.

This file will be modified to announce when it reaches beta release.

