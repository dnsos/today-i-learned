---
layout: default
title: "Posts"
---

<ul>
  <% collections.posts.resources.each do |post| %>
    <li class="border-b border-gray-200">
      <a href="<%= post.relative_url %>" class="w-full p-3 flex justify-between items-center transition-colors hover:bg-gray-100">
        <h2 class="text-2xl"><%= post.data.title %></h2>
        <% puts post.data %>
        <time class="text-base"><%= post.data.date.strftime('%d.%m.%Y') %></time>
      </a>
    </li>
  <% end %>
</ul>
