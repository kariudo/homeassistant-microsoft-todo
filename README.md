# Microsoft To Do integration for Home Assistant

The integration brings your Microsoft To Do tasks into Home Assistant and allows creating new tasks.

This fork ([kariudo/homeassistant-microsoft-todo](https://github.com/kariudo/homeassistant-microsoft-todo)) supports better behavior for overdue tasks, and the ability to get tasks due today (useful for automations with notifications and TTS annoucements etc).

## Work in progress

Work is still in progress and there might be breaking changes. Tasks for the 1st release are tracked on [the MVP board](https://github.com/black-roland/homeassistant-microsoft-todo/projects/2).

## Installation

This component can be installed using [HACS](https://github.com/hacs/integration). Simply add a custom repository `black-roland/homeassistant-microsoft-todo` and install the integration.

Alternatively, that's possible to copy the content of the `custom_components` to [the config directory](https://developers.home-assistant.io/docs/en/creating_component_loading.html).

## Setup

To get access to Microsoft To Do API you need to register an application in Azure:

1. Open [Azure portal](https://portal.azure.com/#home);
1. Go to app registrations:<br />
   ![App registrations](https://share.roland.black/file/black-roland-share/96db74de-fb21-11e9-a480-f81654971495/app-registrations.gif)
1. Register a new personal app and obtain client ID and secret:<br />
   ![App registration and credentials](https://share.roland.black/file/black-roland-share/515dfc34-9bbb-11ea-81d7-50eb7157640d/register-app-obtain-creds.gif)
1. Add a redirect URI: `https://[YOUR HOME ASSISTANT URL:PORT]/api/microsoft-todo`, replace `[YOUR HOME ASSISTANT URL:PORT]` with the domain name and port of our Home Assistant instance:<br />
   ![Redirect URI](https://share.roland.black/file/black-roland-share/d9a41612-fb22-11e9-8f90-f81654971495/redirect-uri.gif)<br />
   To be able to authenticate please make sure that [`base_url` option](https://www.home-assistant.io/integrations/http/#base_url) is configured properly and your browser can access the redirect URI (it doesn’t have to be accessible from the Internet, it can be even `localhost`).

## Configuration

Add the following section to your `configuration.yaml` file:

```yaml
calendar:
  - platform: microsoft_todo
    client_id: YOUR_CLIENT_ID
    client_secret: YOUR_CLIENT_SECRET
```

Restart Home Assistant and finalize authorization through UI. There should be a new configuration request in notifications.

*NOTE*: After successful auth please restart your Home Assistant again (it's known issue #10). 

## Services

To create a task in Microsoft To Do you can call `microsoft_todo.new_task` service.

Simple example:

```yaml
- service: microsoft_todo.new_task
  data:
    subject: "Test task"
    list_name: "Home Assistant"
```

Automation example:

```yaml
automation:
  - alias: "Remind to pay utility bill"
    trigger:
      platform: time
      at: "00:00:00"
    condition:
      condition: template
      value_template: "{{ now().day == 1 }}"
    action:
      - service: microsoft_todo.new_task
        data_template:
          subject: "Pay utility bill for {{ now().replace(month=now().month - 1).strftime('%B') }}" # previous month name
          list_name: "Home Assistant"
          note: "Pay online: http://example.com/pay/"
          due_date: "{{ now().strftime('%Y-%m-09') }}" # due 9th
          reminder_date_time: "{{ now().strftime('%Y-%m-%dT17:00:00') }}" # at 17:00 today
```

## Calendar sensors

The integration creates sensors for each to-do list with tasks exposed as `all_tasks` attribute. You can find sensors on the Developer Tools page prefixed with `calendar.`.

Sensors can be used in automations:

```yaml
automation:
  - alias: "Remind to buy groceries after work"
    trigger:
      platform: state
      entity_id: person.bob
      from: "office"
      to: "away"
    condition:
      condition: template
      value_template: "{{ state_attr('calendar.shopping_list', 'all_tasks') | length > 0 }}"
    action:
      - service: notify.email_bob
        data:
          title: "Shopping list"
          message: "Don't forget to check your shopping list before going home"
```

Or sensors can be used in Lovelace. There are multiple ways to show your tasks: by simply using [ssmorris/lovelace-attribute-list-view](https://github.com/ssmorris/lovelace-attribute-list-view) card or by adding a custom markdown card with Jinja template:

```yaml
type: markdown
content: |-
  {% for task in state_attr('calendar.PUT_YOUR_LIST_NAME_HERE', 'all_tasks') -%}
  - {{ task }}
  {% endfor %}
title: To Do
```

![Markdown card](https://user-images.githubusercontent.com/1756198/106674478-5a6cb080-65c4-11eb-9306-e363ff399f28.png)

### Create Template Sensors

You can add binary sensors linked to the presense of your task, and even wrap your Markdown Card, etc. in a Conditional card as well by leveraging template sensors. This example show show to get a binary sensor from `Tasks Due Today` for a list called "house".

```yaml
template:
  - binary_sensor:
    # Microsoft To Do
    - name: "Tasks Due Today"
      state: >
        {% if state_attr('calendar.house', 'duetoday_tasks') | length > 0 %}
          on
        {% else %}
          off
        {% endif %}
```

The coresponding markdown card might look like:

```yaml
type: conditional
conditions:
  - entity: binary_sensor.tasks_due_today
    state: 'on'
card:
  type: markdown
  content: |-
      {% for task in state_attr('calendar.house', 'duetoday_tasks') -%}
      - {{ task }}
      {% endfor %}
  title: 'Today'
```
