# evennia-alexa
![img](https://img.shields.io/badge/build-alpha-yellow) Feel free to steal this code, but don't trust it.

This makes the server interfaceable with an Alexa device. It functions, but man, Alexa is just total garbo at understanding commands. To be honest, integrating this makes playing for regular users kind of lame, as random people with no username start bouncing in and out of the server constantly. It's really an all-or-nothing thing.

Things to note:
- Look at the cool use of kwargs to send giant packets of useful data to a protocol! Inspiration for the first remarkably good MXP integration, perhaps?
- Different types of interactions need different UI displays. Traversing rooms and checking out your equipment are completely different interactions. The card_type kwarg ties everything together on both the server end (what gets sent?) and the Alexa end (what should I show the user?)
- You kinda have to rewrite your commands to use this, particularly the traverse commands. The Alexa integration sends 'move east, move west' etc. It's up to you to make sure your exits are aliased, and some fuzzy matching on the exit aliases doesn't hurt either, since natural speech remains annoyingly imprecise.

```
Example objects.py

    def msg(self, text=None, **kwargs):
        """
        Generates an Alexa-compatible response, then disconnects the session.
        Replaces self.msg, to some degree. If no text= is passed, it tries to recreate
        it from onscreen_primary, onscreen_secondary, onscreen_tertiary. If that fails,
        it uses the speech variable, which is required.
        """
        from evennia.utils import logger, delay

        logger.log_msg("{text}".format(text=kwargs))
        # don't send any empties

        clean_kwargs = {}
        for k, v in kwargs.items():
            if v:
                clean_kwargs[k] = v
        if text:
            clean_kwargs["text"] = text

        super().msg(**clean_kwargs)
        delay(1, self.disconnect_alexa)

    @property
    def is_alexa_session(self):
        return False

    def disconnect_alexa(self):
        if self.is_alexa_session:
            for session in self.sessions.all():
                self.account.disconnect_session_from_account(session)

```

```
example rooms.py

    def return_appearance(self, looker, **kwargs):
        appearance = {'flags': [], 'name': self.get_display_name(looker), 'desc': self.db.desc or ""}

        visible = [con for con in self.contents if con != looker and con.access(looker, "view")]
        exits = [con for con in visible if con.destination]
        players = [con for con in visible if con.has_account]
        things = [con for con in visible if con not in exits and con not in players]

        appearance['exits'] = [con.get_display_name(looker) for con in exits if con.access(looker, "view")]
        appearance['players'] = [con.get_display_name(looker) for con in players if con.access(looker, "view")]
        appearance['things'] = [con.get_display_name(looker) for con in things if con.access(looker, "view")]
        appearance['card_type'] = 'traverse'

        return appearance
```

```
example character.py


    def msg(self, text=None, from_obj=None, session=None, options=None, **kwargs):
        """
        Generates an Alexa-compatible response, then disconnects the session.
        Replaces self.msg. Documented in Alexa client request_handler.py
        """
        from evennia.utils import logger, delay

        logger.log_msg("{text}".format(text=kwargs))
        # don't send any empties

        clean_kwargs = {}
        for k, v in kwargs.items():
            if v:
                clean_kwargs[k] = v

        if text:
            clean_kwargs["text"] = text

        if self.is_superuser and settings.DEBUG:
            # print debug lines
            for k, v in clean_kwargs.items():
                super().msg("|gDEBUG: |555{}|g : |333{}|n".format(k, v))
            super().msg("|500########################################|n")

        super().msg(**clean_kwargs)
        delay(1, self.disconnect_alexa)

    @property
    def is_alexa_session(self):
        return False

    def disconnect_alexa(self):
        pass

    def return_appearance(self, looker, **kwargs):
        appearance = {'flags': [], 'name': self.get_display_name(looker), 'desc': self.db.desc or ""}
        return appearance

    def at_look(self, target, **kwargs):
        from utils import change_ssml_voice
        if not target.access(self, "view"):
            try:
                self.msg(text="Could not view '%s'." % target.get_display_name(self, **kwargs))
            except AttributeError:
                self.msg(text="Could not view '%s'." % target.key)

        appearance = target.return_appearance(self, **kwargs)
        name = appearance.get("name", "")
        desc = appearance.get("desc", "")
        typeclass = appearance.get("typeclass", None)
        flags = appearance.get("flags", [])
        exits = appearance.get("exits", [])
        players = appearance.get("players", [])
        things = appearance.get("things", [])
        card_type = appearance.get("card_type", 'simple')

        text = f"|[512|555{name}|n"
        card_title = name
        card_text = ""
        best_hint = ""
        card_text_verbose=""

        # display flags
        if flags:
            text += "  [ |n|[500 {} |n ]".format(" ".join(flags).upper())
            card_text_verbose = "  [ {} ]".format(" ".join(flags).upper())

        # display name and desc
        text += f"|/{desc}"
        card_text += f"{desc}"
        # speech_verbose = "$ssml({},voice name=\"Brian\"): {}".format(name, desc)
        speech_verbose = change_ssml_voice(f"{name}", "Justin") + desc
        speech_trunc = "{}".format(desc)
        card_text_verbose += card_text

        # display exits and contents
        exit_str_raw = ", ".join(exits)
        if exits:
            text += "|/|/|nYou can go: |[540|112{}|n".format("|n, |[540|112".join(exits))
            speech_verbose += change_ssml_voice("You can go: {}".format(", ".join(exits)), "Justin")
            card_text_verbose += "|/|/You can go: {}".format(exit_str_raw)
            best_hint = "move {}".format(exits[0])

        thing_str_raw = ", ".join(things)

        if things:
            text += "|/|nYou see: |[002|555{}|n".format("|n, |[002|555".join(things + players))
            speech_verbose += change_ssml_voice("You see: {}".format(", ".join(things)), "Justin")
            card_text_verbose += "|/|/You see: {}".format(thing_str_raw)
            best_hint = "look at {}".format(things[0])

        if self.can_level_up:
            best_hint="level up"
            text += "|/|/***You can level up! Say 'level up'.***"

        #card_title=f"$ssml({card_title},voice naxrme=\"Justin\")"

        self.msg(
            text=text,
            speech_verbose=speech_verbose,
            speech_trunc=speech_trunc,
            card_title=card_title,
            card_text=card_text,
            card_text_verbose=card_text_verbose,
            card_type=card_type,
            exits=exits,
            contents=things,
            hint_text=best_hint
        )

        target.at_desc(looker=self, **kwargs)

class AlexaCharacter(Character):
    def at_object_creation(self):
        super().at_object_creation()
        from commands.default_cmdsets import AlexaCmdSet
        self.cmdset.add(AlexaCmdSet, permanent=True)

    @property
    def is_alexa_session(self):
        return True

    def at_post_puppet(self):
        self.account.db._last_puppet = self

    def disconnect_alexa(self):
        if self.is_alexa_session:
            for session in self.sessions.all():
                self.account.disconnect_session_from_account(session)

```

```
inlinefuncs.py support for changing TTS voice

def ssml(text, tag, *args, **kwargs):
    """
    Usage:
        $ssml(I want to tell you a secret, voice name="Kendra")
        $ssml(I am not a real human, amazon:emotion name="disappointed" intensity="medium")
        $ssml(You won't say anything, will you?, amazon:effect name="whispered")
    """
    session = kwargs.get("session")
    if not session.protocol_flags.get("CLIENTNAME", "") in ["ALEXA"]:
        return text
    if not text or not tag:
        return text
    tag = tag.strip()
    tag_code = tag.split(" ", 1)
    return "<{before}>{text}</{after}>".format(before=tag, text=text, after=tag_code[0])
    
```
