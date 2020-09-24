# kubectl命令行交互

以kubectl create -f xxx.yaml 为例，你会在pkg/kubectl/cmd/create 下找到一个create.go。
类似的，所有kubectl 的命令 都可以在 pkg/kubectl/cmd 下找到。

## kubectl初始化

kubectl初始化，添加各类命令行cobra.Command接口。

Cobra 是一个 Golang 包，它提供了简单的接口来创建命令行程序。

![avatar](picture/cobra.png)

github: https://github.com/spf13/cobra

- 创建了一个工厂对象，用于创建产品，首先生成车间Builder,然后执行DO,生成产品Result
- 根据产品调用visitor执行调用client接口生成request对象
- 添加了所有kubectl的命令对象

```
func NewKubectlCommand(in io.Reader, out, err io.Writer) *cobra.Command {
    // Parent command to which all subcommands are added.
    cmds := &cobra.Command{
        Use:   "kubectl",
        Short: i18n.T("kubectl controls the Kubernetes cluster manager"),
        Long: templates.LongDesc(`
      kubectl controls the Kubernetes cluster manager.

      Find more information at:
            https://kubernetes.io/docs/reference/kubectl/overview/`),
        Run: runHelp,
        BashCompletionFunction: bashCompletionFunc,
    }

    flags := cmds.PersistentFlags()
    flags.SetNormalizeFunc(utilflag.WarnWordSepNormalizeFunc) // Warn for "_" flags

    // Normalize all flags that are coming from other packages or pre-configurations
    // a.k.a. change all "_" to "-". e.g. glog package
    flags.SetNormalizeFunc(utilflag.WordSepNormalizeFunc)

    kubeConfigFlags := genericclioptions.NewConfigFlags()
    kubeConfigFlags.AddFlags(flags)
    matchVersionKubeConfigFlags := cmdutil.NewMatchVersionFlags(kubeConfigFlags)
    matchVersionKubeConfigFlags.AddFlags(cmds.PersistentFlags())

    cmds.PersistentFlags().AddGoFlagSet(flag.CommandLine)

    f := cmdutil.NewFactory(matchVersionKubeConfigFlags)//创建了一个工厂对象，用于创建产品，首先生成车间Builder,然后执行DO,生成产品Result

    // Sending in 'nil' for the getLanguageFn() results in using
    // the LANG environment variable.
    //
    // TODO: Consider adding a flag or file preference for setting
    // the language, instead of just loading from the LANG env. variable.
    i18n.LoadTranslations("kubectl", nil)

    // From this point and forward we get warnings on flags that contain "_" separators
    cmds.SetGlobalNormalizationFunc(utilflag.WarnWordSepNormalizeFunc)

    ioStreams := genericclioptions.IOStreams{In: in, Out: out, ErrOut: err}

    groups := templates.CommandGroups{
        {
            Message: "Basic Commands (Beginner):",
            Commands: []*cobra.Command{
                create.NewCmdCreate(f, ioStreams),
                NewCmdExposeService(f, ioStreams),
                NewCmdRun(f, ioStreams),
                set.NewCmdSet(f, ioStreams),
                deprecatedAlias("run-container", NewCmdRun(f, ioStreams)),
            },
        },
        {
            Message: "Basic Commands (Intermediate):",
            Commands: []*cobra.Command{
                NewCmdExplain("kubectl", f, ioStreams),
                get.NewCmdGet("kubectl", f, ioStreams),
                NewCmdEdit(f, ioStreams),
                NewCmdDelete(f, ioStreams),
            },
        },
        {
            Message: "Deploy Commands:",
            Commands: []*cobra.Command{
                rollout.NewCmdRollout(f, ioStreams),
                NewCmdRollingUpdate(f, ioStreams),
                NewCmdScale(f, ioStreams),
                NewCmdAutoscale(f, ioStreams),
            },
        },
        {
            Message: "Cluster Management Commands:",
            Commands: []*cobra.Command{
                NewCmdCertificate(f, ioStreams),
                NewCmdClusterInfo(f, ioStreams),
                NewCmdTop(f, ioStreams),
                NewCmdCordon(f, ioStreams),
                NewCmdUncordon(f, ioStreams),
                NewCmdDrain(f, ioStreams),
                NewCmdTaint(f, ioStreams),
            },
        },
        {
            Message: "Troubleshooting and Debugging Commands:",
            Commands: []*cobra.Command{
                NewCmdDescribe("kubectl", f, ioStreams),
                NewCmdLogs(f, ioStreams),
                NewCmdAttach(f, ioStreams),
                NewCmdExec(f, ioStreams),
                NewCmdPortForward(f, ioStreams),
                NewCmdProxy(f, ioStreams),
                NewCmdCp(f, ioStreams),
                auth.NewCmdAuth(f, ioStreams),
            },
        },
        {
            Message: "Advanced Commands:",
            Commands: []*cobra.Command{
                NewCmdApply("kubectl", f, ioStreams),
                NewCmdPatch(f, ioStreams),
                NewCmdReplace(f, ioStreams),
                wait.NewCmdWait(f, ioStreams),
                NewCmdConvert(f, ioStreams),
            },
        },
        {
            Message: "Settings Commands:",
            Commands: []*cobra.Command{
                NewCmdLabel(f, ioStreams),
                NewCmdAnnotate("kubectl", f, ioStreams),
                NewCmdCompletion(ioStreams.Out, ""),
            },
        },
    }
    groups.Add(cmds)

    filters := []string{"options"}

    // Hide the "alpha" subcommand if there are no alpha commands in this build.
    alpha := NewCmdAlpha(f, ioStreams)
    if !alpha.HasSubCommands() {
        filters = append(filters, alpha.Name())
    }

    templates.ActsAsRootCommand(cmds, filters, groups...)

    for name, completion := range bash_completion_flags {
        if cmds.Flag(name) != nil {
            if cmds.Flag(name).Annotations == nil {
                cmds.Flag(name).Annotations = map[string][]string{}
            }
            cmds.Flag(name).Annotations[cobra.BashCompCustom] = append(
                cmds.Flag(name).Annotations[cobra.BashCompCustom],
                completion,
            )
        }
    }

    cmds.AddCommand(alpha)
    cmds.AddCommand(cmdconfig.NewCmdConfig(f, clientcmd.NewDefaultPathOptions(), ioStreams))
    cmds.AddCommand(NewCmdPlugin(f, ioStreams))
    cmds.AddCommand(NewCmdVersion(f, ioStreams))
    cmds.AddCommand(NewCmdApiVersions(f, ioStreams))
    cmds.AddCommand(NewCmdApiResources(f, ioStreams))
    cmds.AddCommand(NewCmdOptions(ioStreams.Out))

    return cmds
}       
```

## create命令为例，观察者模式分析

调用NewCmdCreate命令，来为create命令添加自命令以及运行函数RunCreate
当调用create命令时，具体会调用到RunCreate命令
工厂模式，生成车间Builder，配置车间然后Do(),生成产品Result
Result中根据Builder配置信息设置一个重要的成员visitor（有不同类型观察者）
Result.Visit()其实就是Result.visitor.Visitor(fn(Info,err))
不同的观察者visior会生成不同调用对象信息结构体Info
然后将Info作为参数传入到fn函数中
fn函数会根据Info中的RESTClient来调用具体的接口生成Request请求体对象，然后向Api Server发送Request请求

```
func (o *CreateOptions) RunCreate(f cmdutil.Factory, cmd *cobra.Command) error {
    // raw only makes sense for a single file resource multiple objects aren't likely to do what you want.
    // the validator enforces this, so
    if len(o.Raw) > 0 {
        return o.raw(f)
    }

    if o.EditBeforeCreate {
        return RunEditOnCreate(f, o.PrintFlags, o.RecordFlags, o.IOStreams, cmd, &o.FilenameOptions)
    }
    schema, err := f.Validator(cmdutil.GetFlagBool(cmd, "validate"))
    if err != nil {
        return err
    }

    cmdNamespace, enforceNamespace, err := f.ToRawKubeConfigLoader().Namespace()
    if err != nil {
        return err
    }

    r := f.NewBuilder().
        Unstructured().
        Schema(schema).
        ContinueOnError().
        NamespaceParam(cmdNamespace).DefaultNamespace().
        FilenameParam(enforceNamespace, &o.FilenameOptions).
        LabelSelectorParam(o.Selector).
        Flatten().
        Do()//工厂模式，生成车间，中间部分的操作均为配置车间，然后Do(),生成产品Result
    err = r.Err()
    if err != nil {
        return err
    }

    count := 0
    err = r.Visit(func(info *resource.Info, err error) error {//观察者生成Info，调用func回调函数
        if err != nil {
            return err
        }
        if err := kubectl.CreateOrUpdateAnnotation(cmdutil.GetFlagBool(cmd, cmdutil.ApplyAnnotationsFlag), info.Object, cmdutil.InternalVersionJSONEncoder()); err != nil {
            return cmdutil.AddSourceToErr("creating", info.Source, err)
        }

        if err := o.Recorder.Record(info.Object); err != nil {
            glog.V(4).Infof("error recording current command: %v", err)
        }

        if !o.DryRun {//这里的DryRun代表只创建不实际运行
            //这里实现根据info中的RESTClient接口来生成对应的Request请求体来请求API Server
            if err := createAndRefresh(info); err != nil {
                return cmdutil.AddSourceToErr("creating", info.Source, err)
            }
        }

        count++

        return o.PrintObj(info.Object)
    })
    if err != nil {
        return err
    }
    if count == 0 {
        return fmt.Errorf("no objects passed to create")
    }
    return nil
}
```
