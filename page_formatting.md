# Message Formatting

All text in Slack uses the same system of escaping: chat messages, direct messages, file comments, etc.


## URLs and Escaping

Messages can contain any displayable Unicode sequence of characters (all messages must use UTF-8 encoding), 
but need to be slightly escaped by clients. Before sending messages to the server, clients should make the
following transforms:

    & replaced with &amp;
    < replaced with &lt;
    > replaced with &gt;

For example:

    User types     : Hello & <world>
    Sent to server : Hello &amp; &lt;world&gt;

This is done so that messages can contain special escaped sequences. For example, to refer to a channel or
user within a message, a client should send the following messages:

    Why not join <#C024BE7LR|general>?

    Hey <@U024BE7LH|bob>, did you see my file?

These escape sequences are then forwarded to all members of the channel as usual, and clients can format
these links specially. The readable name can be included after the ID, by separating them with a pipe (`|`) 
character. Both of these formats are valid:

    <@U024BE7LH>
    <@U024BE7LH|bob>

For regular URL links, clients should just include the URL in the message; It is not the responsibility of 
individual clients to detect URLs within typed messages. For example, the following messages can be sent 
to the server:

    This message contains a URL http://foo.com/

    So does this one: www.foo.com

URL detection is performed by the server. We do this so that URL detection (a non-trivial operation) is
performed consistently across multiple clients. The example messages will be broadcast to other clients 
as follows:

    This message contains a URL <http://foo.com/>

    So does this one: <http://www.foo.com|www.foo.com>

In the first case, the URL is detected as-is. In the second, the full URL is included first, then a pipe 
character (`|`), then finally the originally typed version of the URL.

The server will also detect mailto links in the format `<email|user>`.  For example:

    <mailto:bob@example.com|Bob>

Some messages contain special words with extra behavior. For these we use the
format `<!foo>`, where `foo` is a special command. Currently defined commands
are `!channel`, `!group`, `!here`, and `!everyone` (group is just a synonym for channel
- both can be used in channels and groups). These indicate an @channel, @group, @here,
or @everyone message, and should cause a notification to be displayed by the
client.

Note that to display @here on older mobile clients you will need to specify a label
with the `<!here>` command (eg. `<!here|here>`).

More commands will be added in the future. Commands that are not recognised
can be output as-is, with the enclosing brackets to indicate something about
the text is special. If a label is present it should be used. For example:

 * `<!foo>` should be displayed as `<foo>`
 * `<!foo|label>` should be displayed as `<label>`

To display messages in a web-client, the client should take the following
steps:

1.  Detect all sequences matching `<(.*?)>`
2.  Within those sequences, format content starting with `#C` as a channel link
3.  Within those sequences, format content starting with `@U` as a user link
4.  Within those sequences, format content starting with `!` according to the rules for the special command.
5.  For remaining sequences, format as a link
6.  Once the format has been determined, check for a pipe - if present, use the text following the pipe as the link label

Because the ampersands and angled brackets are already escaped, no further translation need take place 
(for a web-client). The server ensures that no extra un-escaped angled brackets or ampersands are 
included in the message.

When a client sends a message, the response that is returned by the server contains the server-formatted
version of the message. Clients should use this to replace their own local version of the message so that
urls are correctly highlighted.


## Parsing modes

By default, messages you pass to API methods and webhooks will be assumed to be pre-formatted based on
the above spec. That is, you can include marked up URLs, user links, channel links and commands, but 
we will still linkify any non-linked URLs present in your message.

    IN  : Foo <!everyone> bar http://test.com
    OUT : Foo <!everyone> bar <http://test.com>

By default, Slack will not linkify channel names (starting with a '#') and usernames (starting with
an '@'). You can enable this behavior by passing `link_names=1` as an argument. This behavior is always 
enabled in `parse=full` mode (see below).

    IN  : Hello @bob, say hi to @everyone in #general
    OUT : Hello <@U123|bob>, say hi to <!everyone> in <#C1234|general>

If you don't want Slack to perform _any_ processing on your message, pass an argument of `parse=none`.

    IN  : Foo <!everyone> bar http://test.com
    OUT : Foo <!everyone> bar http://test.com

(In this case, Slack will still test the validity of your markup - we won't send invalid messages to clients).

If you want Slack to treat your message as completely unformatted, pass `parse=full`. This will ignore 
any markup formatting you added to your message.

    IN  : Foo <!everyone> bar http://test.com
    OUT : Foo &lt;!everyone&gt; bar <http://test.com>

In full parse mode, we will find and linkify URLs, channel names (starting with a '#') and usernames (starting 
with an '@').


## Multiline Messages

You can post multiline messages through Slack's APIs. Insert a newline by including the characters `\n` in your text. For example:

    {
        "text": "This is a line of text.\nAnd this is another one."
    }

The above JSON payload will be displayed in the channel as:

![Screenshot of a webhook with a newline](/img/api/incoming_simple.png)

## Emoji

Slack attempts to normalize emoji across multiple platforms using the following approach:

1. All emoji sent to Slack (as chat message to the message server, or arguments to the API) are translated into
   the common 'colon' format (e.g. `:smile:`)
2. At display time, Slack clients are encouraged to convert these colon-sequences into native Emoji where available,
   otherwise fallback to images.

The Slack message server and API handle conversion from several binary emoji formats - the Unicode Unified format
(used by OSX 10.7+ and iOS 6+), the Softbank format (used by iOS 5) and the Google format (used by some Android
devices). These Unicode code points will be converted into their colon-format equivalents.

The list of emoji supported are taken from [https://github.com/iamcal/emoji-data](https://github.com/iamcal/emoji-data)


## Message Attachments

You can create richly-formatted messages using Attachments. **[Learn how to add attachments to messages.](/docs/attachments)**


## Message Formatting

Slack messages may be formatted using a simple markup language similar to [Markdown](https://daringfireball.net/projects/markdown/). Supported formatting includes: `` ```pre``` ``, `` `code` ``, `_italic_`, and `*bold*`; full details are available on our [help site](https://slack.zendesk.com/hc/en-us/articles/202288908-How-can-I-add-formatting-to-my-messages-).

By default bot message text will be formatted, but [attachments](/docs/attachments) are not. To disable formatting on a non-user message, set the `mrkdwn` property to `false`. To enable formatting on attachment fields, set the `mrkdwn_in` array on each attachment to the list of fields to process. Some examples:

    {
        "text": "*bold* `code` _italic_",
        "username": "markdownbot",
        "mrkdwn": true
    }

    {
        "text": "*not bold*",
        "username": "markdownbot",
        "mrkdwn": false
    }

    {
        "attachments": [
            {
                "title": "Title",
                "pretext": "Pretext _supports_ mrkdwn",
                "text": "Testing *right now!*",

                "mrkdwn_in": ["text", "pretext"]
            }
        ]
    }

Valid values for `mrkdwn_in` are: `["pretext", "text", "fields"]`. Setting `"fields"` will enable markup formatting for the `value` of each field.
