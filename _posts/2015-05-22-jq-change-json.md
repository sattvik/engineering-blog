---
layout: post
title: "Modifying JSON on the command line"
subtitle: "Using jq to change a value"
header-img: "img/mon-field_rows.jpg"
author: "Jessica Kerr"
githubProfile : "jessitron"
twitterHandle : "jessitron"
avatarUrl : "https://avatars3.githubusercontent.com/u/1149737?v=3"
tags: [jq, json, aws, tutorials]
---

<style scoped>
  .projectKey { color: red }
  .map { color: orange }
  .plus { color: brown }
  .newValue { color: blue }
  .dot { color: hotpink }
  .toEntries { color: purple }
  .fromEntries { color: green }
</style>

If you need to parse through some JSON data at the command line, [`jq`](http://stedolan.github.io/jq/) is here for you.
`jq` is its own programming language. There are tons of examples of how to use `jq` to extract data from JSON; 
this post shows how we use it to modify JSON.

[AWS CloudFormation](http://aws.amazon.com/cloudformation/) turns a JSON stack definition (plus a JSON configuration file)
into a whole interconnected bunch of AWS resources. I frequently want to update my configuration. 
Using `jq`, I can do this from the command line. That means I can script it for automated tests.

The configuration file looks like this:


<div class="highlight"><pre><code class="language-json" data-lang="json">[{
  "ParameterKey": "{{ "Project" | sc: "projectKey" }}",
  "ParameterValue": "&lt;changeMe&gt;"
 }, 
 {
  "ParameterKey": "DockerInstanceType",
  "ParameterValue": "m3.medium"
}]
</code></pre></div>

The JSON is an array of objects, each with ParameterKey and ParameterValue. I want to change the ParameterValue for {{ "a particular ParameterKey" | sc: "projectKey" }}. Here's the syntax:


<div class="highlight"><pre><code class="language-bash" data-lang="bash">cat config.json | 
  jq '{{ "map" | sc: "map" }}(if .ParameterKey == "{{ "Project" | sc: "projectKey" }}"
          then {{"."|sc:"dot"}} {{"+"|sc:"plus"}} <span class="newValue">{"ParameterValue"="jess-project"}</span>
          else {{"."|sc:"dot"}}
          end
         )' > populated_config.json
</code></pre></div>

This says, "{{ "For each object in the array" | sc: "map" }}:
check if ParameterKey is "{{"Project"|sc:"projectKey"}}". If so,
{{"combine"|sc:"plus"}} {{ "that object"|sc:"dot"}}
with {{"this other one"|sc:"newValue"}} (right-hand-side values win, so my ParameterValue overrides the existing one). If not, leave {{"the object"|sc:"dot"}} alone." 
The output file now contains

<div class="highlight"><pre><code class="language-bash" data-lang="bash">[{
   "ParameterKey": "{{"Project"|sc:"projectKey"}}",
   "ParameterValue": "{{"jess-project"|sc:"newValue"}}"
 },
 {
   "ParameterKey": "DockerInstanceType",
   "ParameterValue": "m3.medium"
}]
</code></pre></div>

Hooray! The `jq` {{"map"|sc:"map"}} function, combined with a conditional, let me change a particular value.

Since I do this often, and I'm on a Mac, I made a crude bash function and threw it in my [.bash_profile](http://web.physics.ucsb.edu/~pcs/apps/bash/intro-bash.html) so it will always be available:

<div class="highlight"><pre><code class="language-bash" data-lang="bash">function populate-config() { 
  jq "{{"map"|sc:"map"}}(if .ParameterKey == \"{{"$1"|sc:"projectKey"}}\" 
          then . + {\"ParameterValue\":\"{{"$2"|sc:"newValue"}}"} 
          else . 
          end)"
}
</code></pre></div>

Now I can say

<div class="highlight"><pre><code class="language-bash" data-lang="bash">cat config.json | 
  populate-config {{ "Project" | sc: "projectKey" }} {{"jess-project"|sc:"newValue"}} |
  populate-config {{"DockerInstanceType"|sc:"projectKey"}} {{"t2.micro"|sc:"newValue"}} > populated_config.json
</code></pre></div>

Piping to the `populate-config` function over and over lets me change multiple values.

CAUTION:

The `jq` {{"map"|sc:"map"}} function works on arrays. It's easy in this config file, because the JSON happens to be an array. If instead I'm changing a value within an object, such as:

<div class="highlight"><pre><code class="language-bash" data-lang="bash">{
  "{{"honesty"|sc:"projectKey"}}": "Apple Jack",
  "laughter": "Pinkie Pie",
  "loyalty": "Rainbow Dash"
}
</code></pre></div>

then I must {{"convert the object's properties to an array"|sc:"toEntries"}}, {{"map"|sc:"map"}} over that array and then {{"convert the array back into an object"|sc:"fromEntries"}}.


<div class="highlight"><pre><code class="language-bash" data-lang="bash">cat ponies.json | 
  jq '{{"to_entries"|sc:"toEntries"}} | 
       {{"map"|sc:"map"}}(if .key == "{{"honesty"|sc:"projectKey"}}" 
          then . + {"value":"{{"Trixie"|sc:"newValue"}}"} 
          else . 
          end
         ) | 
      {{"from_entries"|sc:"fromEntries"}}'
</code></pre></div>

this gives:

<div class="highlight"><pre><code class="language-bash" data-lang="bash">{
  "{{"honesty"|sc:"projectKey"}}": "{{"Trixie"|sc:"newValue"}}",
  "laughter": "Pinkie Pie",
  "loyalty": "Rainbow Dash"
}
</code></pre></div>

That sneaky Trixie. Try running `cat ponies.json | jq to_entries` to see how this works.

FURTHER INVESTIGATION:

You might ask, how do I modify arrays of objects nested inside other objects? If you find the answer, please ping us on twitter [@MonPlatformEng](http://twitter.com/MonPlatformEng) because I'm wondering this too.
