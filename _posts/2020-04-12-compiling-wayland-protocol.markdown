---
layout: post
title:  "Compiling Wayland Protocol"
date:   2021-04-12 17:30:14 -0300
categories: wayland protocol
image: "/assets/img/2021-04-12-Compiling-Wayland-Protocol.png"
share-img: "/assets/img/2021-04-12-Compiling-Wayland-Protocol.png"
author: Felipe Magno de Almeida
---
We're developing our own Wayland Compositor, and we noticed most
projects just use libwayland-server. However, we want more control
over the on-the-write protocol I/O and better integration with our C++
codebase.

If you've written a Wayland Compositor you know that the protocol is
defined in XML format. So, we decided to create our compiler for the
Wayland protocols.

In these protocol definitions there are interfaces with requests and
events defined. These are the most important abstractions for
communication between a Wayland Compositor and its clients.

# Protocol

You can see this example from wayland.xml from 1.0 version:

{% highlight XML %}
  <interface name="wl_display" version="1">
    <description summary="core global object">
      The core global object.  This is a special singleton object.  It
      is used for internal Wayland protocol features.
    </description>

    <request name="get_registry">
      <description summary="get global registry object">
	This request creates a registry object that allows the client
	to list and bind the global objects available from the
	compositor.

	It should be noted that the server side resources consumed in
	response to a get_registry request can only be released when the
	client disconnects, not when the client side proxy is destroyed.
	Therefore, clients should invoke get_registry as infrequently as
	possible to avoid wasting memory.
      </description>
      <arg name="registry" type="new_id" interface="wl_registry"
	   summary="global registry object"/>
    </request>
{% endhighlight %}

As you can see, a request called `get_registry` is defined with only one
single argument of type `new_id`.

The `get_registry` is one of the most important requests a client
makes right in the beginning, to be able to get the `wl_registry`,
which the client can listen on events to get other global objects and
which it can use to bind client objects to the compositor.

Since the protocol doesn't define return values from requests, the way
to create objects is to have the client define ids for them in
getters. That's what the type `new_id` does.

The client calls `get_registry` and passes a id, usually 0, to be set
as the id of the registry that it is supposedly getting.

Now the client knows that the registry id is 0, since it got it with
that argument.

The `get_registry` request also triggers the global event for each
global object the wayland compositor has and needs to publish to the
client.

# Compiler

The [VWM](https://github.com/expertisesolutions/vwm) Wayland
Compositor has a compiler that reads multiple XML wayland protocol and
creates a single class that handles requests and translates events to
the on-the-write binary protocol of the wayland.

The protocol is quite simple and is implemented in our compiled result
as simple POD copies over the write and special handling for strings
and file descriptor arguments.

For example, the result of the `get_registry` request is compiled as:

{% highlight C++ %}
case interface_::wl_display:
  switch (op)
  {
    case 1:{
      std::cout << "op get_registry\n";
      struct values0
      {
        std::uint32_t arg0;
      };
      unsigned offset = 0;
      struct values0 values0;
      if constexpr (!std::is_empty<struct values0>::value)
      {
        std::memcpy(&values0, payload.data() + offset, sizeof(values0));
        offset += sizeof(values0);
      }
      std::uint32_t & arg0 = values0.arg0;
      this->wl_display_get_registry(object, arg0);
      break;}
    default: break;
  };
{% endhighlight %}

The constant `1` represents the index of the request in the interface
`wl_display`. The `server_protocol` class has one member function that
interprets the message to a request or event and calls the appropriate
function by using CRTP, aka Curiously Recurring Template Parameter, to
customize the Wayland Compositor behavior. Which is where the
`wl_display_get_registry` is defined.

As we can see in `client.hpp:464` where `wl_display_get_registry` is implemented:

{% highlight C++ %}
void wl_display_get_registry (object& obj, uint32_t new_id)
{
  std::cout << "wl_display_get_registry with new id " << new_id << std::endl;

  add_object (new_id, {vwm::wayland::generated::interface_::wl_registry});
  server_protocol().wl_registry_global (new_id, 1,  "wl_compositor", 4);
  server_protocol().wl_registry_global (new_id, 2,  "wl_subcompositor", 1);
  server_protocol().wl_registry_global (new_id, 2,  "wl_data_device_manager", 3);
  server_protocol().wl_registry_global (new_id, 3,  "wl_shm", 1);
  server_protocol().wl_registry_global (new_id, 4,  "wl_drm", 2);
  server_protocol().wl_registry_global (new_id, 5,  "wl_seat", 5);
  server_protocol().wl_registry_global (new_id, 6,  "wl_output", 3);
  server_protocol().wl_registry_global (new_id, 7,  "xdg_wm_base", 2);
  server_protocol().wl_registry_global (new_id, 8,  "wl_shell", 1);
  server_protocol().wl_registry_global (new_id, 9,  "zwp_linux_dmabuf_v1", 3);
  server_protocol().wl_registry_global (new_id, 10, "zwp_linux_explicit_synchronization_v1", 2);
}
{% endhighlight %}

# Conclusion

As you can see, taking control of the protocol and handling is
possible and a small part of the work needed to write a Wayland
Compositor. It is also a rewarding task that allows extending the
Wayland protocol to other mediums beside UNIX sockets. The biggest
challenge is supporting `shm_buffer` over slower mediums.

This also allows implementing faster and more scalable protocol
handling code.

