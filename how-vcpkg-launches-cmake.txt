How VCPKG launches CMake
========================

The command used to launch CMake is located in `vcpkg/toolsrc/src/vcpkg/build.cpp` in function `vcpkg::Build::do_build_package`.
In this function we can find this code:

```
    const std::string cmd_launch_cmake = System::make_cmake_cmd(
            cmake_exe_path,
            paths.ports_cmake,
            {
                {"CMD", "BUILD"},
                {"PORT", config.scf.core_paragraph->name},
                {"CURRENT_PORT_DIR", config.port_dir},
                {"TARGET_TRIPLET", spec.triplet().canonical_name()},
                {"VCPKG_PLATFORM_TOOLSET", toolset.version.c_str()},
                {"VCPKG_USE_HEAD_VERSION",
                 Util::Enum::to_bool(config.build_package_options.use_head_version) ? "1" : "0"},
                {"_VCPKG_NO_DOWNLOADS", !Util::Enum::to_bool(config.build_package_options.allow_downloads) ? "1" : "0"},
                {"_VCPKG_DOWNLOAD_TOOL", to_string(config.build_package_options.download_tool)},
                {"GIT", git_exe_path},
                {"FEATURES", Strings::join(";", config.feature_list)},
                {"ALL_FEATURES", all_features},
            });


 - `cmake_exe_path` is the path to the CMake installed exe that will be used.
 - `paths.ports_cmake` is the path to the directory containing all the CMake scripts for each of the packages.
 - `config.scf.core_paragraph->name` - NOT SURE YET.
 - `spec.triplet().canonical_name()` is the name of one of the triplets supported, named like the CMake scripts in the `vcpkg/triplets` directory.
    These files contain mainly assignation of `vcpkg`-specific CMake variables, so they need to be set before building.
 - `toolset.version.c_str()` the toolset to use, I believe these are the ones listed in `cmake --help`.
 - `config.build_package_options.use_head_version` - NOT SURE YET.
 - `config.build_package_options.allow_downloads` is true if vcpkg is allowed to download dependencies.
 - `git_exe_path` is the path to git's installation (so it's a requirement that git be installed).
 - `config.feature_list` and `all_features` - NOT SURE YET.


The function `System::make_cmake_cmd` is defined in `vcpkg/toolsrc/src/vcpkg/system.cpp` as:

    std::string make_cmake_cmd(const fs::path& cmake_exe,
                               const fs::path& cmake_script,
                               const std::vector<CMakeVariable>& pass_variables)
    {
        const std::string cmd_cmake_pass_variables = Strings::join(" ", pass_variables, [](auto&& v) { return v.s; });
        return Strings::format(
            R"("%s" %s -P "%s")", cmake_exe.u8string(), cmd_cmake_pass_variables, cmake_script.generic_u8string());
    }

Here `CMakeVariable` is a simple struct with special constructors, doing:

    CMakeVariable::CMakeVariable(const CStringView varname, const char* varvalue)
        : s(Strings::format(R"("-D%s=%s")", varname, varvalue))
    {
    }

where `s` is a `std::string`.

Therefore, the command should look like this:

    path/to/cmake -DVARIABLE_A=value_a -DVARIABLE_B=value_b [...] -P path/to/vcpkg/ports/

However, this is executed depending on other environment variables set before this line is executed:

    auto command = make_build_env_cmd(pre_build_info, toolset);
    if (!command.empty())
    {
    #ifdef _WIN32
        command.append(" & ");
    #else
        command.append(" && ");
    #endif
    }
    command.append(cmd_launch_cmake);
    // ...
    const int return_code = System::cmd_execute_clean(command);

Here `cmd_execute_clean` focuses on launching the command processes correctly.
`make_build_env_cmd` setup the necessary environement variables:





