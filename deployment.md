# Deployment

## CI/CD

### Build specific version 
To build a project without changing the version numbers manually, the following steps are needed:

1. Create a `BuildCommon.targets` file with specific task to achieve this
2. Import the new file in the project

This will allow the following execution.
```batch
msbuild /p:VersionAssembly=<version> /p:Configuration=Release <project-name>.sln
```

> more info: [Lev Gimelfarb](http://www.lionhack.com/2014/02/13/msbuild-override-assembly-version/)

#### Create the file `BuildCommon.targets`
Inside the solution folder, create a file called `BuildCommon.targets` with the following content

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <!--
        Defining custom Targets to execute before project compilation starts.
    -->
    <PropertyGroup>
        <CompileDependsOn>
            CommonBuildDefineModifiedAssemblyVersion;
            $(CompileDependsOn);
        </CompileDependsOn>
    </PropertyGroup>

    
    <!--
        Creates modified version of AssemblyInfo.cs, replaces [AssemblyVersion] attribute with the one specifying actual build version (from MSBuild properties), and includes that file instead of the original AssemblyInfo.cs in the compilation.
    -->
    <Target Name="CommonBuildDefineModifiedAssemblyVersion" Condition="'$(VersionAssembly)' != ''">
        <!-- Find AssemblyInfo.cs in the "Compile" Items. Remove it from "Compile" Items because we will use a modified version instead. -->
        <ItemGroup>
            <OriginalAssemblyInfo Include="@(Compile)" Condition="%(Filename) == 'AssemblyInfo' And %(Extension) == '.cs'" />
            <Compile Remove="**/AssemblyInfo.cs" />
        </ItemGroup>
        <!-- Copy the original AssemblyInfo.cs to obj\ folder, i.e. $(IntermediateOutputPath). The copied filepath is saved into @(ModifiedAssemblyInfo) Item. -->
        <Copy SourceFiles="@(OriginalAssemblyInfo)"
            DestinationFiles="@(OriginalAssemblyInfo->'$(IntermediateOutputPath)%(Identity)')">
            <Output TaskParameter="DestinationFiles" ItemName="ModifiedAssemblyInfo"/>
        </Copy>
        <!-- Replace the version bit (in AssemblyVersion and AssemblyFileVersion attributes) using regular expression. Use the defined property: $(VersionAssembly). -->
        <Message Text="Setting AssemblyVersion to $(VersionAssembly)" />
        <RegexUpdateFile Files="@(ModifiedAssemblyInfo)"
                    Regex="Version\(&quot;(\d+)\.(\d+)(\.(\d+)\.(\d+)|\.*)&quot;\)"
                    ReplacementText="Version(&quot;$(VersionAssembly)&quot;)"
                    />
        <!-- Include the modified AssemblyInfo.cs file in "Compile" items (instead of the original). -->
        <ItemGroup>
            <Compile Include="@(ModifiedAssemblyInfo)" />
        </ItemGroup>
    </Target>

    <UsingTask TaskName="RegexUpdateFile" TaskFactory="CodeTaskFactory" AssemblyFile="$(MSBuildToolsPath)\Microsoft.Build.Tasks.v4.0.dll">
        <ParameterGroup>
            <Files ParameterType="Microsoft.Build.Framework.ITaskItem[]" Required="true" />
            <Regex ParameterType="System.String" Required="true" />
            <ReplacementText ParameterType="System.String" Required="true" />
        </ParameterGroup>
        <Task>
            <Reference Include="System.Core" />
            <Using Namespace="System" />
            <Using Namespace="System.IO" />
            <Using Namespace="System.Text.RegularExpressions" />
            <Using Namespace="Microsoft.Build.Framework" />
            <Using Namespace="Microsoft.Build.Utilities" />
            <Code Type="Fragment" Language="cs">
                <![CDATA[
                    try {
                        var rx = new System.Text.RegularExpressions.Regex(this.Regex);
                        for (int i = 0; i < Files.Length; ++i)
                        {
                            var path = Files[i].GetMetadata("FullPath");
                            if (!File.Exists(path)) continue;

                            var txt = File.ReadAllText(path);
                            txt = rx.Replace(txt, this.ReplacementText);
                            File.WriteAllText(path, txt);
                        }
                        return true;
                    }
                    catch (Exception ex) {
                        Log.LogErrorFromException(ex);
                        return false;
                    }
                ]]>
            </Code>
        </Task>
    </UsingTask>
</Project>
```

#### Import the new file in the `.csproj` file
Add the following line before closing the `Project` tag:

```xml
<Import Project="$(SolutionDir)\BuildCommon.targets" />
```

It should look like this
```xml
<Project>
    ...
    .csproj content
    ...
  <ItemGroup>
    <Folder Include="Models\" />
  </ItemGroup>
  <Import Project="$(MSBuildToolsPath)\Microsoft.CSharp.targets" />
  <Import Project="$(SolutionDir)\BuildCommon.targets" />
</Project>
```