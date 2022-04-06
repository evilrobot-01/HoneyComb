# Patching Mesa

As per https://gist.github.com/jnettlet/1f461487bee9c3e2a2d994f25441717d, create a `/etc/drirc`file with the following contents (restarting X to take effect):

    <driconf>
        <!-- Please always enable app-specific workarounds for all drivers and
             screens. -->
        <device>
            <application name="XWayland" executable="Xwayland">
                <option name="mesa_extension_override" value="-GL_ARB_buffer_storage" />
            </application>
            <application name="Xorg" executable="Xorg">
                <option name="mesa_extension_override" value="-GL_ARB_buffer_storage" />
            </application>
        </device>
    </driconf>
