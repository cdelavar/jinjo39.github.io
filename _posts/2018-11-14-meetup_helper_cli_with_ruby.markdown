---
layout: post
title:      "Meetup Helper CLI with Ruby"
date:       2018-11-14 16:42:13 +0000
permalink:  meetup_helper_cli_with_ruby
---

## Woot, Meetup Helper! Meetup.com == social life

The [meetup_helper](https://github.com/jinjo39/MeetupHelper) command-line interface is a way to find the history of past meetups that a member of meetup.com has attended. I decided to write this program for my first project because meetup.com is the number 1 reason I have a social life, and the site does not make it easy to find past history data or download pictures from meetups. Currently, if you want to find the past events you’ve rsvp’d yes to, you would have to navigate to each separate group page, click on the past meetups link, and manually find all of the meetups you’ve attended. You’d also have to do a similar manual procedure, right clicking and saving one by one, if you wanted to download pictures from a meetup you attended. Meetup Helper allows you to download all of the pictures and get details from meetups you’ve attended in one fell swoop.

## Ruby Mechanize is Your Friend

Meetup Helper utilizes the Mechanize ruby gem in order to fill out the user login fields and sign in to meetup.com. The Mechanize gem uses Nokogiri to parse a HTML page for its forms and buttons and allows you to enter and submit the values for fields. The `sign_in` class method for the `MeetupHelper::SignIn` class stores a new instance of mechanize in a class variable, `@@agent`; this allows the instance of mechanize to scrape the meetup.com login page, and we can set the second `Form` object, `page.forms[1]`, equal to our sign_in:

```bash
sign_in = page.forms[1]
puts "Sign in to Meetup.com"
puts "Email: "
sign_in.email = gets.chomp
puts "Password: "
sign_in.password = STDIN.noecho(&:gets).chomp
```

The user is prompted for their login email and password which is hidden from the terminal (you need to add require "io/console" to hide the password). The form is then submitted with `@@agent.submit(sign_in)`. A reader method for the class variable `@@agent`, which is our authenticated login to the meetup site, will be used to connect to the Meetup API.`::ApiConnect.connect_to_meetup_api` will then use the authenticated login to navigate to the page with the user’s api key. The page is scraped for the user’s api key and member id with the css method:

```bash
@@api_key = MeetupHelper::SignIn.agent.get(url).css("#api-key-reveal").first.attribute("value").text
@@member_id = MeetupHelper::SignIn.agent.get(url).css("#nav-account-links a").attribute("href").value.split("/")[-1]
```

## I :heart: the meetup_client gem
Meetup Helper utilizes the meetup_client gem to connect to the Meetup API.
The user’s api key is then used to configure the meetup_client initializer, which will allow calls to the Meetup API to be made:

```bash
MeetupClient.configure do |config|
  config.api_key = @@api_key
end
```

The meetup_client gem requires api calls to be made using a hash with request parameters outlined on the meetup [api page](https://www.meetup.com/meetup_api/). I decided to initialize the session with past meetup event history because it’s the most laborious to extract from the site manually, so after the user signs in, new `Meetup` objects are instantiated from this data:

```bash
params = {rsvp: 'yes', member_id: MeetupHelper::ApiConnect.member_id, status: 'past', fields: "photo_album_id”}
```

The params are called with a new instance of MeetupApi.new, courtesy of the meetup_client gem, which will allow for the call to the Meetup API and return a json containing a list of past events the user has rsvp’d to. I included a line of code to change the key values from strings to symbols, just to make defining the `Meetup` objects values easier later on:

```bash
@@results = JSON.parse(events.to_json, {:symbolize_names => true} )
```

## Grabbing that Data
The json hash is then parsed and iterated through each event to make `Meetup` objects with attributes, including details of the meetup event and group it belongs to:

```bash
def self.parse_events
  MeetupHelper::ApiCalls.results[:results].each do |event|
    meetup = MeetupHelper::Meetup.new
    meetup.event_name = event[:name]
    meetup.group_id = event[:group][:id]
    meetup.group_name = event[:group][:name]
    meetup.event_id = event[:id]
    event_date = event[:time]
    meetup.date = Time.at(event_date/1000)
    meetup.venue = event[:venue]
    meetup.yes_rsvps = event[:yes_rsvp_count]
    meetup.photo_album_id = event[:photo_album_id]
  end
end
```

After the data for each meetup is parsed, the user is prompted with options to display the data and download pictures:

```bash
class MeetupHelper::Options
  def self.list_options
    puts <<-DOC
      1. List meetup groups from attended events
      2. List past meetups attended by group
      3. Delete meetups not attended
      4. Get pictures from meetups
      5. List all meetups attended
      6. How many meetups you've attended
      7. Exit
    DOC
  end
end
```

The easiest way to use this cli is to enter `1` for a list of the meetup groups you've attended, and then to copy the name of that group in order to enter it when prompted during the other options, such as to list all the past meetups attended or to download pictures. The reason for this is because the ruby code that searches for the group name is case sensitive. You can also "delete" events that you've rsvp'd to, but did not actually attend; this does not delete the event history from the website, but you can use this feature to virtually delete the events and find a sum total of events you've actually attended.

## Grabbing Photos
Another call to the Meetup API must be made in order to download photos from past events. This call is similar to the first made for past events, but requires the photos method to be called with the photo_album_id param, whose value was parsed during the first call and is readily available as a `Meetup` object attribute: 

```bash
params = {photo_album_id: photo_id}

def call_api_photos(params = nil)
  photos = @meetup_api.photos(params)
  @@results = JSON.parse(photos.to_json, {:symbolize_names => true} )
  MeetupHelper::Parser.parse_photos
end
```
This method is called within the `Meetup.get_pictures_from_event` method.

## Mechanize saves the day again
Calling the API yields a json of urls and detail attributes for the pictures in the event's photo album. The hash is then iterated though in order to collect only the photo urls, labeled as `highres_link`, in an array. A new Mechanize agent is instantiated, and the `pluggable_parser` method from the `Mechanize::Download` class must be called on the agent in order to download the photos. Once that's done, the photos array can be iterated though, and each photo from the event will be saved to the path provided by the user. Admittedly, this feature was the hardest and took the longest for me to implement, but I was overjoyed when I finally got it to work:

```bash
def self.parse_photos
  photos = MeetupHelper::ApiCalls.results[:results].collect { |photophoto[:highres_link]}
  agent = Mechanize.new
  agent.pluggable_parser.default = Mechanize::Download
  puts "Directory path for saving the files:"
  path = gets.strip
  photos.each do |photo|
    agent.get(photo).save(File.join(path, "#{File.basename(photo)}"))
  end
end
```
In the future, I plan to implement a database in order to save the meetup event data locally. Unfortunately, if any of the meetup groups I'm a part of get cancelled, all event history would be deleted, and I will not have access to the data. Having that event history is nice because then I can remember all the cool places I've gone hiking and camping and all the random things I've done.






