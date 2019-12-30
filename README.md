WeeWX Extension for sending your data to a custom REST API
===

*DISCLAIMER: I'm not responsible to anything that happens to your weather station, or WeeWX installation or anything in between. This is just a bit of code I added to my WeeWX installation as I wanted to send the data that was sent to Weather Underground to a custom API that I built.*

# Steps to make this work:

1) install WeeWX
2) Make sure you're currently sending data to any sort of destination (like WU or AWEKAS)
3) include the following code in your "restx.py" file (depending on the distro where you installed can be in different locations. If you don't know where the file is just run the following command "cd / && find | grep restx.py" without quotes):

```
class StdCustomApi(StdRESTful):
    """Specialized version of the Ambient protocol for your custom API.
    """

    def __init__(self, engine, config_dict):

        super(StdCustomApi, self).__init__(engine, config_dict)

        _ambient_dict = get_site_dict(
            config_dict, 'CustomAPI', 'station', 'password')
        if _ambient_dict is None:
            return

        _essentials_dict = search_up(config_dict['StdRESTful']['CustomAPI'], 'Essentials', {})

        syslog.syslog(syslog.LOG_DEBUG, "restx: CustomAPI essentials: %s" % _essentials_dict)

        # Get the manager dictionary:
        _manager_dict = weewx.manager.get_manager_dict_from_config(
            config_dict, 'wx_binding')

        # The default is to not do an archive post if a rapidfire post
        # has been specified, but this can be overridden
        do_rapidfire_post = to_bool(_ambient_dict.pop('rapidfire', False))
        do_archive_post = to_bool(_ambient_dict.pop('archive_post',
                                                    not do_rapidfire_post))

        # Get the URL of the custom API
        apiURLConfig = _ambient_dict.pop('apiURL')

        if do_archive_post:
            _ambient_dict.setdefault('server_url', apiURLConfig)
            self.archive_queue = Queue.Queue()
            self.archive_thread = AmbientThread(
                self.archive_queue,
                _manager_dict,
                protocol_name="CustomAPI",
                essentials=_essentials_dict,
                **_ambient_dict)
            self.archive_thread.start()
            self.bind(weewx.NEW_ARCHIVE_RECORD, self.new_archive_record)
            syslog.syslog(syslog.LOG_INFO, "restx: CustomAPI: "
                                           "Data for station %s will be posted" %
                          _ambient_dict['station'])

        if do_rapidfire_post:
            _ambient_dict.setdefault('server_url', apiURLConfig)
            _ambient_dict.setdefault('log_success', False)
            _ambient_dict.setdefault('log_failure', False)
            _ambient_dict.setdefault('max_backlog', 0)
            _ambient_dict.setdefault('max_tries', 1)
            _ambient_dict.setdefault('rtfreq',  2.5)
            self.cached_values = CachedValues()
            self.loop_queue = Queue.Queue()
            self.loop_thread = AmbientLoopThread(
                self.loop_queue,
                _manager_dict,
                protocol_name="CustomAPI",
                essentials=_essentials_dict,
                **_ambient_dict)
            self.loop_thread.start()
            self.bind(weewx.NEW_LOOP_PACKET, self.new_loop_packet)
            syslog.syslog(syslog.LOG_INFO, "restx: CustomAPI: "
                                           "Data for station %s will be posted" %
                          _ambient_dict['station'])

    def new_loop_packet(self, event):
        """Puts new LOOP packets in the loop queue"""
        if weewx.debug >= 3:
            syslog.syslog(syslog.LOG_DEBUG, "restx: raw packet: %s" % to_sorted_string(event.packet))
        self.cached_values.update(event.packet, event.packet['dateTime'])
        if weewx.debug >= 3:
            syslog.syslog(syslog.LOG_DEBUG, "restx: cached packet: %s" %
                          to_sorted_string(self.cached_values.get_packet(event.packet['dateTime'])))
        self.loop_queue.put(
            self.cached_values.get_packet(event.packet['dateTime']))

    def new_archive_record(self, event):
        """Puts new archive records in the archive queue"""
        self.archive_queue.put(event.record)
```

4) include the following code in your weewx.conf file (same as above):
Section [StdRESTful]

```
    [[CustomAPI]]
        # This section is for configuring posts to your own custom REST API.

        # If you wish to do this, set the option 'enable' to true,
        # and specify a station (e.g., 'KORHOODR3') and password, if you have any.
        # To guard against parsing errors, put the password in quotes.
        enable = false
        apiURL= "http://myCustomApiURL"
        station  = replace_me
        password = "replace_me"

        # Set the following to True to have weewx pull the values every one second and hit your API with them
        # Not all hardware can support it. See the User's Guide.
        rapidfire = False
```

Section [Engine]
```
restful_services = weewx.restx.StdStationRegistry, weewx.restx.StdWunderground, weewx.restx.StdPWSweather, weewx.restx.StdCWOP, weewx.restx.StdWOW, weewx.restx.StdAWEKAS, weewx.restx.StdCustomAPI
```

In the [[CustomAPI]] section in weewx.conf you'll be able to include the address of the api you want to send the info to. As I didn't tinker much with how the platform sends the data, it is currently sending the data through a GET or POST request (I forgot the method) and values as parameters in a querystring (yup, you figured it out)

Enjoy!