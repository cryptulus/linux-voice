ronn = find_program('ronn', required: get_option('man'))
if ronn.found()
  custom_target('man',
    input : files('linux-voice.8.ronn'),
    output : 'linux-voice.8',
    command : [ronn, '--output-dir', '@OUTDIR@', '--roff', '@INPUT@'],
    install : true, install_dir: join_paths(get_option('mandir'),'man8'))
endif
