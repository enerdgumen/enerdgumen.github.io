title: Monitoring logs with Sauron
date: 2013-02-13 12:21
tags: python, system

[Sauron](http://github.com/maurocchi/sauron) is a simple self-contained *Python* 2.3 script, without any dependency, useful to monitor errors on log files in real time from the command-line.

By default it follows all log files in `/var/log` directory and shown the rows containing an "error" word (bad, cannot, error, exception, fail, fails, failed, failure, incorrect, invalid, unable, unexpected, wrong).

    $ sauron [options] [paths (default /var/log)]

    Options:
        -h, --help            show this help message and exit
        -f FILENAME_REGEX, --files=FILENAME_REGEX
                              filename regex (default 'log$')
        -a, --all             the log file are followed from the begin
        -w WORDS, --words=WORDS
                              comma-separed words which filters the lines (default:
                              error, exception, fail, fails, failed, failure,
                              cannot, unable, unexpected, incorrect, invalid, bad,
                              wrong)
        -r LINE_REGEX, --regex=LINE_REGEX
                              regex to filter the log lines (it overwrites the
                              --words option)
        -i CHECK_INTERVAL, --interval=CHECK_INTERVAL
                              interval in seconds before to scan again the paths
                              (default 1)
        --version
