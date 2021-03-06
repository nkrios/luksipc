Testing
=======
It's always a good idea to perform some tests before you give some unknown tool
by some unknown author a shot to handle your precious data. Here's a hint how
you could do this. I've setup some garbage partition on a drive which is
exactly 1234 MiB in size (1293942784 bytes). That partition is filled
completely with zeros::

    # dd if=/dev/zero of=/dev/sda1 bs=1M
    dd: error writing ‘/dev/sda1’: No space left on device
    1235+0 records in
    1234+0 records out
    1293942784 bytes (1,3 GB) copied, 30,6571 s, 42,2 MB/s

Let's check that the pattern matches::

    # md5sum /dev/sda1
    e83b40511b7b154b1816ef4c03d6be7d  /dev/sda1
    
    # dd if=/dev/zero bs=1M count=1234 | md5sum
    1234+0 records in
    1234+0 records out
    1293942784 bytes (1,3 GB) copied, 2,71539 s, 477 MB/s
    e83b40511b7b154b1816ef4c03d6be7d  -

Alright, so the partition is **really** filled with 1234 MiB of zeros.

Let's LUKSify it::

    # luksipc -d /dev/sda1
    WARNING! luksipc will perform the following actions:
       => Normal LUKSification of plain device /dev/sda1
       -> luksFormat will be performed on /dev/sda1
    
    Please confirm you have completed the checklist:
        [1] You have resized the contained filesystem(s) appropriately
        [2] You have unmounted any contained filesystem(s)
        [3] You will ensure secure storage of the keyfile that will be generated at /root/initial_keyfile.bin
        [4] Power conditions are satisfied (i.e. your laptop is not running off battery)
        [5] You have a backup of all important data on /dev/sda1
    
        /dev/sda1: 1234 MiB = 1.2 GiB
        Chunk size: 10485760 bytes = 10.0 MiB
        Keyfile: /root/initial_keyfile.bin
        LUKS format parameters: None given
    
    Are all these conditions satisfied, then answer uppercase yes: YES
    [I]: Generated raw device alias: /dev/sda1 -> /dev/mapper/alias_luksipc_raw_944e8f9034a6344f
    [I]: Size of reading device /dev/sda1 is 1293942784 bytes (1234 MiB + 0 bytes)
    [I]: Performing dm-crypt status lookup on mapper name 'luksipc_41c33f9940708688'
    [I]: Performing luksFormat of raw device /dev/mapper/alias_luksipc_raw_944e8f9034a6344f using key file /root/initial_keyfile.bin
    [I]: Performing luksOpen of raw device /dev/mapper/alias_luksipc_raw_944e8f9034a6344f using key file /root/initial_keyfile.bin and device mapper handle luksipc_41c33f9940708688
    [I]: Size of writing device /dev/mapper/luksipc_41c33f9940708688 is 1291845632 bytes (1232 MiB + 0 bytes)
    [I]: Write disk smaller than read disk, 2097152 bytes occupied by LUKS header (2048 kB + 0 bytes)
    [I]: Starting copying of data, read offset 10485760, write offset 0
    [I]:  0:00:   8.9%       110 MiB / 1232 MiB    44.9 MiB/s   Left:    1122 MiB  0:00 h:m
    [I]:  0:00:  17.0%       210 MiB / 1232 MiB    43.2 MiB/s   Left:    1022 MiB  0:00 h:m
    [I]:  0:00:  25.2%       310 MiB / 1232 MiB    18.8 MiB/s   Left:     922 MiB  0:00 h:m
    [I]:  0:00:  33.3%       410 MiB / 1232 MiB    21.3 MiB/s   Left:     822 MiB  0:00 h:m
    [I]:  0:00:  41.4%       510 MiB / 1232 MiB    23.4 MiB/s   Left:     722 MiB  0:00 h:m
    [I]:  0:00:  49.5%       610 MiB / 1232 MiB    22.0 MiB/s   Left:     622 MiB  0:00 h:m
    [I]:  0:00:  57.6%       710 MiB / 1232 MiB    18.7 MiB/s   Left:     522 MiB  0:00 h:m
    [I]:  0:00:  65.7%       810 MiB / 1232 MiB    19.8 MiB/s   Left:     422 MiB  0:00 h:m
    [I]:  0:00:  73.9%       910 MiB / 1232 MiB    20.3 MiB/s   Left:     322 MiB  0:00 h:m
    [I]:  0:00:  82.0%      1010 MiB / 1232 MiB    17.8 MiB/s   Left:     222 MiB  0:00 h:m
    [I]:  0:00:  90.1%      1110 MiB / 1232 MiB    18.6 MiB/s   Left:     122 MiB  0:00 h:m
    [I]:  0:01:  98.2%      1210 MiB / 1232 MiB    19.4 MiB/s   Left:      22 MiB  0:00 h:m
    [I]: Disk copy completed successfully.
    [I]: Synchronizing disk...
    [I]: Synchronizing of disk finished.

Then luksOpen it::

    # cryptsetup luksOpen /dev/sda1 myluksdev -d /root/initial_keyfile.bin

And check the hash::

    # md5sum /dev/mapper/myluksdev
    e2226de7d184a3c9bd4c1e3d8a56b1b2  /dev/mapper/myluksdev

The hash value differs from what it said before - this is absolutely to be
expected! The reason for this is that the device is now shorter (because part
of the space is used for the 2 MiB LUKS header). Proof::

    # dd if=/dev/zero bs=1M count=1232 | md5sum
    1232+0 records in
    1232+0 records out
    1291845632 bytes (1,3 GB) copied, 2,6588 s, 486 MB/s
    e2226de7d184a3c9bd4c1e3d8a56b1b2  -

Now let's check the current key and reLUKSify it with a different key and
algorithm! First, let's check out the "before" values::

    # dmsetup table myluksdev --showkeys
    0 2523136 crypt aes-xts-plain64 d164b3fd2b7d482fc6e0a2d0e58f51c5dafe4560507322cb29af4bd8f552ba4f 0 8:1 4096
    
    # cryptsetup luksDump /dev/sda1
    LUKS header information for /dev/sda1
    
    Version:        1
    Cipher name:    aes
    Cipher mode:    xts-plain64
    Hash spec:      sha1
    Payload offset: 4096
    MK bits:        256
    MK digest:      dd 08 3d 43 ae 50 64 7c d9 c6 20 cb de dd 7a 62 69 10 63 fe
    MK salt:        f1 95 eb 18 2d 90 61 e9 c8 df 4b 4d 44 ab 62 87
                    5a f5 39 5a c4 f5 3b 7a 09 8c f1 75 33 a5 f3 25
    MK iterations:  50375
    UUID:           127277bf-b07b-4209-bf55-37cb1c10c83b
    
    Key Slot 0: ENABLED
        Iterations:             201892
        Salt:                   fc d9 3a 73 b4 73 ee 98 6c 35 34 a0 c7 7d 8a 71
                                5b 75 b7 6c 75 af 65 20 eb 90 7c 69 34 10 1e a6
        Key material offset:    8
        AF stripes:             4000
    Key Slot 1: DISABLED
    Key Slot 2: DISABLED
    Key Slot 3: DISABLED
    Key Slot 4: DISABLED
    Key Slot 5: DISABLED
    Key Slot 6: DISABLED
    Key Slot 7: DISABLED

Then reLUKSify::

    # my /root/initial_keyfile.bin /root/initial_keyfile_old.bin
    
    # luksipc -d /dev/sda1 --readdev /dev/mapper/myluksdev --luksparams='-c,twofish-lrw-benbi,-s,320,-h,sha256'
    WARNING! luksipc will perform the following actions:
       => reLUKSification of LUKS device /dev/sda1
       -> Which has been unlocked at /dev/mapper/myluksdev
       -> luksFormat will be performed on /dev/sda1
    
    Please confirm you have completed the checklist:
        [1] You have resized the contained filesystem(s) appropriately
        [2] You have unmounted any contained filesystem(s)
        [3] You will ensure secure storage of the keyfile that will be generated at /root/initial_keyfile.bin
        [4] Power conditions are satisfied (i.e. your laptop is not running off battery)
        [5] You have a backup of all important data on /dev/sda1
    
        /dev/sda1: 1234 MiB = 1.2 GiB
        Chunk size: 10485760 bytes = 10.0 MiB
        Keyfile: /root/initial_keyfile.bin
        LUKS format parameters: -c,twofish-lrw-benbi,-s,320,-h,sha256
    
    Are all these conditions satisfied, then answer uppercase yes: YES
    [I]: Generated raw device alias: /dev/sda1 -> /dev/mapper/alias_luksipc_raw_c84651981fc98f36
    [I]: Size of reading device /dev/mapper/myluksdev is 1291845632 bytes (1232 MiB + 0 bytes)
    [I]: Performing dm-crypt status lookup on mapper name 'luksipc_9afeee69aec4912c'
    [I]: Performing luksFormat of raw device /dev/mapper/alias_luksipc_raw_c84651981fc98f36 using key file /root/initial_keyfile.bin
    [I]: Performing luksOpen of raw device /dev/mapper/alias_luksipc_raw_c84651981fc98f36 using key file /root/initial_keyfile.bin and device mapper handle luksipc_9afeee69aec4912c
    [I]: Size of writing device /dev/mapper/luksipc_9afeee69aec4912c is 1291845632 bytes (1232 MiB + 0 bytes)
    [I]: Write disk size equal to read disk size.
    [I]: Starting copying of data, read offset 10485760, write offset 0
    [I]:  0:00:   8.9%       110 MiB / 1232 MiB    43.1 MiB/s   Left:    1122 MiB  0:00 h:m
    [I]:  0:00:  17.0%       210 MiB / 1232 MiB    42.0 MiB/s   Left:    1022 MiB  0:00 h:m
    [I]:  0:00:  25.2%       310 MiB / 1232 MiB    28.3 MiB/s   Left:     922 MiB  0:00 h:m
    [I]:  0:00:  33.3%       410 MiB / 1232 MiB    19.1 MiB/s   Left:     822 MiB  0:00 h:m
    [I]:  0:00:  41.4%       510 MiB / 1232 MiB    21.3 MiB/s   Left:     722 MiB  0:00 h:m
    [I]:  0:00:  49.5%       610 MiB / 1232 MiB    21.6 MiB/s   Left:     622 MiB  0:00 h:m
    [I]:  0:00:  57.6%       710 MiB / 1232 MiB    19.9 MiB/s   Left:     522 MiB  0:00 h:m
    [I]:  0:00:  65.7%       810 MiB / 1232 MiB    18.6 MiB/s   Left:     422 MiB  0:00 h:m
    [I]:  0:00:  73.9%       910 MiB / 1232 MiB    19.8 MiB/s   Left:     322 MiB  0:00 h:m
    [I]:  0:00:  82.0%      1010 MiB / 1232 MiB    19.4 MiB/s   Left:     222 MiB  0:00 h:m
    [I]:  0:01:  90.1%      1110 MiB / 1232 MiB    17.6 MiB/s   Left:     122 MiB  0:00 h:m
    [I]:  0:01:  98.2%      1210 MiB / 1232 MiB    18.4 MiB/s   Left:      22 MiB  0:00 h:m
    [I]: Disk copy completed successfully.
    [I]: Synchronizing disk...
    [I]: Synchronizing of disk finished.

Now, let's detach the mapping of the old LUKS container first (this container
now contains complete garbage)::

    # cryptsetup luksClose myluksdev

And reopen it with the correct key::

    # cryptsetup luksOpen /dev/sda1 mynewluksdev -d /root/initial_keyfile.bin

Check that the content is still the same::

    # cat /dev/mapper/mynewluksdev | md5sum
    e2226de7d184a3c9bd4c1e3d8a56b1b2  -

It sure is. Now look at the luksDump output::

    LUKS header information for /dev/sda1
    
    Version:        1
    Cipher name:    twofish
    Cipher mode:    lrw-benbi
    Hash spec:      sha256
    Payload offset: 4096
    MK bits:        320
    MK digest:      10 b9 35 7b c8 23 d7 c3 2a b9 3e e6 95 74 cf 7f ef 75 1b 32
    MK salt:        3f 58 e6 1e 29 e1 c7 a2 f1 14 9e 1f c7 09 fa 23
                    93 7c 9c 59 20 67 d7 a7 7e 7d fe a0 12 9f 0f 25
    MK iterations:  29000
    UUID:           1dd5e426-9e37-4d1e-a6f9-17aa4179eb1e
    
    Key Slot 0: ENABLED
        Iterations:             117215
        Salt:                   9d 58 5c 30 2b dc 35 33 19 bf 78 ab 3e aa 6e 8a
                                fa 6c 9b ee 45 f7 db 9e f1 ab 0c fb cb 3c eb 51
        Key material offset:    8
        AF stripes:             4000
    Key Slot 1: DISABLED
    Key Slot 2: DISABLED
    Key Slot 3: DISABLED
    Key Slot 4: DISABLED
    Key Slot 5: DISABLED
    Key Slot 6: DISABLED
    Key Slot 7: DISABLED

And the used key internally::

    # dmsetup table mynewluksdev --showkeys
    0 2523136 crypt twofish-lrw-benbi d6b007ce62de58b62331f800edf5864da390eb274b908506b368035e7a0f8ea1c3583c2b939928c3 0 8:1 4096

As you can see, completely different keys, completely different algorithm --
but still identical data. It worked :-)

Of course you can do this test with arbitrary data (not just constant zeros). I
was just too lazy to write a PRNG that outputs easily reproducible results.
Feel free to play around with it and please report any and all bugs if you find
some.
