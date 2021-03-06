系统：Windows 10
引擎：Unity 2017.2.1f1

（1）场景布局如下：
运行前：
 ![1](.\pic\1.png)

运行后：
 ![2](.\pic\2.png)

（2）脚本代码：

AssetBundlesEditor.cs

```
using UnityEngine;
using UnityEditor;
using System.IO;

public class AssetBundlesEditor : Editor
{
    [MenuItem("AssetBundles/Build AB for Android")]
    static void BuildAB()
    {
        string dir = Application.streamingAssetsPath;
        if (!Directory.Exists(dir))
        {
            Directory.CreateDirectory(dir);
        }
        BuildPipeline.BuildAssetBundles(dir, BuildAssetBundleOptions.None, BuildTarget.Android);
        AssetDatabase.SaveAssets();
        AssetDatabase.Refresh();
        Debug.LogWarning("Build AB succeeded");
    }

    [MenuItem("AssetBundles/Build Game for Android")]
    static void BuildGame()
    {
        BuildPlayerOptions buildPlayerOptions = new BuildPlayerOptions();
        buildPlayerOptions.scenes = new[] { "Assets/Scenes/Main.unity" };
        buildPlayerOptions.locationPathName = "../package/assetbundle_test.apk";
        buildPlayerOptions.target = BuildTarget.Android;
        buildPlayerOptions.options = BuildOptions.None;

        string strResult = BuildPipeline.BuildPlayer(buildPlayerOptions);
        if (string.IsNullOrEmpty(strResult))
        {
            Debug.LogWarning("Build Game succeeded");
        }
        else
        {
            Debug.LogWarning("Build Game failed: " + strResult);
        }
    }
}
```

 AssetBundlesManager.cs

```
using UnityEngine;
using UnityEngine.Networking;
using UnityEngine.UI;
using System.Collections;
using System.Collections.Generic;
using System.Text;

public class AssetBundlesManager : MonoBehaviour
{
    // 远程路径（服务器地址或该资源的完整路径）
    private StringBuilder _remotePath = new StringBuilder();
    // 本地路径
    private StringBuilder _localPath = new StringBuilder();
    // ab清单
    private AssetBundleManifest _abManifest = null;
    // <ab挂载的对象名, ab对象数组>
    private Dictionary<string, Object[]> _abRootDict = new Dictionary<string, Object[]>();
    // 已加载的ab名字列表
    private List<string> _abList = new List<string>();

    // Text
    public Text _text = null;

    IEnumerator Start()
    {
        // 初始化Text
        _text.text = "";

        // 初始化路径
        _localPath.Append(Application.streamingAssetsPath + "/");
        if (Application.platform == RuntimePlatform.Android)
        {
            _remotePath.Append(_localPath);
        }
        else
        {
            _remotePath.Append(@"file:///" + _localPath);
        }
        // 打印各种路径
        this.Print("_localPath：" + _localPath);
        this.Print("_remotePath：" + _remotePath);

        // 读取manifest文件
        AssetBundle manifestAB = AssetBundle.LoadFromFile(_localPath + "StreamingAssets");
        _abManifest = manifestAB.LoadAsset<AssetBundleManifest>("AssetBundleManifest");

        // 加载公共资源
        yield return LoadAB("material.ab");
        // 加载指定资源
        yield return LoadAB("cube.ab", "GORoot");
        yield return LoadAB("mainui.ab", "UIRoot");
        // 实例化
        InstanceAB();
    }

    // 加载指定资源
    IEnumerator LoadAB(string abName, string rootName = "")
    {
        if (!IsLoaded(abName))
        {
            // AssetBundle资源路径
            string uri = _remotePath + abName;
            this.Print("uri：" + uri);

            // UnityWebRequest
            UnityWebRequest request = UnityWebRequest.GetAssetBundle(uri);
            yield return request.SendWebRequest();

            // 得到AssetBundle对象
            AssetBundle ab = DownloadHandlerAssetBundle.GetContent(request);

            // 读取AssetBundle里面的所有资源
            Object[] objs = ab.LoadAllAssets();
            if (!string.IsNullOrEmpty(rootName))
            {
                _abRootDict.Add(rootName, objs);
            }

            // 加载依赖资源
            string[] depends = _abManifest.GetAllDependencies(abName);
            foreach (string subName in depends)
            {
                if (IsLoaded(subName))
                    continue;

                // 打印指定资源的依赖AssetBundle资源
                string sub_uri = _localPath + subName;
                this.Print("sub：" + sub_uri);
                // 根据依赖继续加载AssetBundle资源
                AssetBundle sub_ab = AssetBundle.LoadFromFile(sub_uri);
                if (sub_ab != null)
                {
                    sub_ab.LoadAllAssets();
                }
            }
        }
    }

    // 实例化
    void InstanceAB()
    {
        foreach (KeyValuePair<string, Object[]> dict in _abRootDict)
        {
            foreach (Object o in dict.Value)
            {
                GameObject go = (GameObject)Instantiate(o); // 实例化对象，根据对象的类型进行转换
                go.transform.SetParent(GameObject.Find(dict.Key).transform);
                go.transform.localPosition = Vector3.zero;
            }
        }
    }

    // 去重-过滤重复加载
    bool IsLoaded(string abName)
    {
        bool isLoaded = false;
        foreach (string listItem in _abList)
        {
            if (listItem.Equals(abName))
            {
                isLoaded = true;
                break;
            }
        }
        if (!isLoaded)
        {
            _abList.Add(abName);
        }

        return isLoaded;
    }

    // 打印
    void Print(string str)
    {
        Debug.Log(str);
        if (_text.text != "")
            _text.text = _text.text + "\n\n";
        _text.text = _text.text + str;
    }
}
```

（3）真机效果：
 ![3](.\pic\3.png)



注意：单独加载纹理ab文件，在Android中是没法显示该纹理，需要把纹理附加在材质球，并通过材质球的加载而加载。



以上简单回顾。

参考资料：

unity2017以上版本的Assetbundle打包
https://blog.csdn.net/elineSea/article/details/79866758

Unity3d 5.x AssetBundle打包与加载
https://www.cnblogs.com/lan-yt/p/7787290.html

Unity中如何使用AssetBundle打包
http://gad.qq.com/article/detail/287854

Unity官方文档-BuildPipeline.BuildPlayer
https://docs.unity3d.com/ScriptReference/BuildPipeline.BuildPlayer.html
