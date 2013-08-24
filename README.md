# dve - the distributed video encoder

This is a small script to do distributed, high quality video encoding.

The script: 

- breaks input video into chunks.
- distributes chunks to different servers via SSH.
- encodes those chunks in parallel.
- reassembles the chunks into final encoded video.

Why do this? So you can encode video using the best settings possible,
and use as many machines as you have available to ensure it doesn't
take forever. ☺

## Usage

By default, dve will just use your local host for encoding, which isn't likely to
improve performance. At a bare minimum, you should specify more than one host
to encode with:

    dve -l host1,host2,host3 media/test.mp4

If you're using a statically linked ffmpeg binary (recommended), then you'll also
want to specify the path to that binary:

    dve -e ~/bin/ffmpeg -l host1,host2,host3 media/test.mp4

After the encoding is completed and the chunks stitched back together, you
should end up with an output file named "test.mp4_new.mkv" in your current working
directory. You can adjust output naming, but note that the output container format
will currently always be mkv:

    dve -s .encoded.mkv -e ~/bin/ffmpeg -l host1,host2,host3 media/test.mp4

dve automatically copies the local ffmpeg to remote systems for encoding, but this
won't work if you want to mix architectures / OSes. If you ensure
there's a valid ffmpeg install in the $PATH on each remote system, you can
disable copying over the local encoder:

    dve -d -l win1,win2,lin1,lin2 media/test.mp4

This allows you to mix Windows/cygwin and Linux hosts on the same
encoding job.

## Benchmarks

Hosts used for this benchmark were dual Xeon L5520 systems with 24GB of RAM,
16 HT cores per host. Input video file is a 4k resolution (4096x2304) test
clip, 3:47 in length.

### ffmpeg on a single host

```bash
$ time nice -n 10 ./ffmpeg -y -v error -stats -i test.mp4 -c:v libx264 -crf 20.0 -preset medium -c:a libvorbis -aq 5 -f matroska test.mkv
frame= 5459 fps=7.4 q=-1.0 Lsize=  530036kB time=00:03:47.43 bitrate=19091.2kbits/s    
real    12m17.177s
user    182m57.340s
sys     0m36.240s
```

### dve with 3 hosts

```bash
$ time dve -o "-c:v libx264 -crf 20.0 -preset medium -c:a libvorbis -aq 5" -l c1,c2,c3 test.mp4
Creating chunks to encode

Computers / CPU cores / Max jobs to run
1:local / 2 / 1

Computer:jobs running/jobs completed/%of started jobs/Average seconds to complete
ETA: 1s 1left 1.57avg  local:1/7/100%/1.6s
Running parallel encoding jobs

Computers / CPU cores / Max jobs to run
1:c1 / 16 / 1
2:c2 / 16 / 1
3:c3 / 16 / 1

Computer:jobs running/jobs completed/%of started jobs/Average seconds to complete
ETA: 380s 6left 64.00avg  c1:1/1/40%/132.0s  c2:1/0/20%/0.0s  c3:1/1/40%/132.0s
Computer:jobs running/jobs completed/%of started jobs
ETA: 90s 2left 45.33avg  1:1/2/37%/138.0s  2:0/2/25%/138.0s  3:1/2/37%/138.0s
Computer:jobs running/jobs completed/%of started jobs/Average seconds to complete
ETA: 42s 1left 42.14avg  c1:0/3/37%/99.7s  c2:0/2/25%/149.5s  c3:1/2/37%/149.5s
Computer:jobs running/jobs completed/%of started jobs
ETA: 50s 1left 50.29avg  1:0/3/37%/118.7s  2:0/2/25%/178.0s  3:1/2/37%/178.0s
Combining chunks into final video file
Cleaning up temporary working files

real    6m17.075s
user    1m29.630s
sys     0m22.697s
```

### Summary

dve has overhead, due to breaking the source file into chunks, transferring those chunks
across the network, retrieving the encoded chunks, and recombining into a new file.

Given these limitations, a ~2x speed increase by using 3 encoding machines is
a reasonable improvement over using a single system.

If you've got benchmarks using more hosts, please submit them!

## Installation

### SSH

SSH is used by GNU parallel to distribute the jobs to target systems.
It's recommended that you use "ssh-keygen" and "ssh-copy-id" to
setup key based authentication to all your remote hosts.

### Pre-reqs

The following need to be installed on the host running this script:

- [ffmpeg](http://johnvansickle.com/ffmpeg/)
- [GNU parallel](https://www.gnu.org/software/parallel/)

It will automatically copy the encoder binary to the target systems and run
it there, but you will need to make sure any required shared libraries are installed
on the target system.

It's recommended that you use the latest ffmpeg statically linked binaries (above),
which will also improve your chances of transcoding new / strange video formats.

### bash

If you're using the statically linked ffmpeg binaries in ~/bin,
you'll need to ensure bash adds ~/bin to your $PATH for non-interactive shells.
Including the following at the top of your ~/.bashrc should be sufficient:

```bash
# User dependent .bashrc file

# Set PATH so it includes user's private bin if it exists
if [ -d "${HOME}/bin" ] ; then
  PATH="${HOME}/bin:${PATH}"
fi
```

### Windows

dve can be run on Windows via [cygwin](http://www.cygwin.com/).

To do so, you'll need to:

- build GNU parallel manually from source (requires make).
- install a static build of [ffmpeg for Windows](http://ffmpeg.zeranoe.com/builds/).
- install (or symlink) above into your $PATH, usually ~/bin.

You'll also need to do the following if you want to use the host to render with:

- [configure sshd](http://www.noah.org/ssh/cygwin-sshd.html)
- alter ~/.bashrc as mentioned above.

## Restrictions

- currently only generates mkv containers on output.

## ⚠ Known Issues

### chunk sizing

Sometimes the last chunk to be split out will be too small to be properly encoded,
and this will cause the whole encode to fail when the pieces are joined at the end
of the job.

Currently the only solution is to adjust the -m and -c options so that the final
chunk of the file is reasonably large.

### ffmpeg / libav
Ubuntu ships the libav fork version ffmpeg wrapper instead of the actual ffmpeg
release. This version is missing many necessary features, including the
[concat demuxer](https://www.ffmpeg.org/faq.html#How-can-I-concatenate-video-files_003f)
used to stitch encoded chunks back together.

For now, you should download and use the statically linked ffmpeg
binaries listed above.

For more background about this fork, see:
[The FFmpeg/Libav situation](http://blog.pkh.me/p/13-the-ffmpeg-libav-situation.html)

## License
dve is copyright 2013 by Graeme Humphries <graeme@sudo.ca>.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see the
[GNU licenses page](http://www.gnu.org/licenses/).
