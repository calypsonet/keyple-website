+++
widget = "blank"  # See https://sourcethemes.com/academic/docs/page-builder/
headless = true  # This file represents a page section.
active = true  # Activate this widget? true/false
weight = 10  # Order that this section will appear.

title = "Keyple Extensions"
subtitle = "List of all available Keyple extensions"

[design]
  # Choose how many columns the section has. Valid values: 1 or 2.
  columns = "1"
  
[design.background]
  # Text color (true=light or false=dark).
  text_color_light = false
  
[design.spacing]
  # Customize the section spacing. Order is top, right, bottom, left.
  padding = ["20px", "0", "20px", "0"]
+++

<table id="extensionsTable">
  <tr class="header">
    <th>Name</th>
    <th>Technology</th>
    <th>Platform</th>
    <th>Target</th>
    <th>Repository</th>
    <th>Documentation</th>
    <th>Latest Release</th>
    <th>Developer</th>
    <th>Star</th>
  </tr>
  <tr>
    <td>Keyple Calypso API - Java</td>
    <td>Calypso®</td>
    <td>PC, MAC, Android</td>
    <td>JVM</td>
    <td><a href="https://github.com/eclipse/keyple-java/">GitHub</a></td>
    <td><a href="/docs/">Developer Guide</a> <br> <a href="/docs/api-reference/">API Reference</a></td>
    <td><a class="js-github-release" href="https://github.com/eclipse/keyple-java/releases" data-repo="eclipse/keyple-java"><!-- V --></a></td>
    <td><a href="https://keyple.org" title="Keyple project"> Keyple project</a></td>
    <td><span style="text-shadow: none;"><a class="github-button" href="https://github.com/eclipse/keyple-java/" data-icon="octicon-star" data-size="large" data-show-count="true" aria-label="Star this on GitHub">Star</a><script async defer src="https://buttons.github.io/buttons.js"></script></span></td>
  </tr>
  <tr>
    <td>Keyple Calypso API - C++</td>
    <td>Calypso®</td>
    <td>PC, MAC</td>
    <td>Native C++</td>
    <td><a href="https://github.com/eclipse/keyple-cpp/">GitHub</a></td>
    <td><a href="/docs/">Developer Guide</a> <br> <a href="/docs/api-reference/">API Reference</a></td>
    <td><a class="js-github-release" href="https://github.com/eclipse/keyple-cpp/releases" data-repo="eclipse/keyple-cpp"><!-- V --></a></td>
    <td><a href="https://keyple.org" title="Keyple project"> Keyple project</a></td>
    <td><span style="text-shadow: none;"><a class="github-button" href="https://github.com/eclipse/keyple-cpp/" data-icon="octicon-star" data-size="large" data-show-count="true" aria-label="Star this on GitHub">Star</a><script async defer src="https://buttons.github.io/buttons.js"></script></span></td>
  </tr>
</table>

{{% alert note %}}
To add an extension send your information to [user mailing list](https://accounts.eclipse.org/mailing-list/keyple-user) or made a [Pull request](https://github.com/eclipse/keyple-website/pulls) contributions on GitHub.
{{% /alert %}}