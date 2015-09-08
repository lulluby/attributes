using System;
using System.Collections.Generic;
using System.Linq;
using System.Reflection;
using System.Runtime.InteropServices;
using System.Text;
using System.Threading.Tasks;

namespace MethodOf
{
    public class MyActionParams
    {
    }

    public class MyActionParamsSample : MyActionParams
    {
    }

    public delegate void MyAction(MyActionParams prms);

    class CheckAccess : Attribute
    {
        /// <summary>
        /// Access key for member. if null or "" - default 
        /// namespace.class[.member]
        /// will be used
        /// </summary>
        public string AccessKey { get; private set; }

        public CheckAccess(string accessKey = null)
        {
            this.AccessKey = accessKey;
        }
    }

    public static class AccessUtils
    {

        private static Dictionary<string, AccessDictionaryItem> _accessRights;
        private static Dictionary<string, List<MemberInfo>> _checkables;

        private static Dictionary<MemberInfo, AccessDictionaryItem> _memberRights;



        static AccessUtils()
        {
            _accessRights = new Dictionary<string, AccessDictionaryItem>();
            _checkables = new Dictionary<string, List<MemberInfo>>();
            _memberRights = new Dictionary<MemberInfo, AccessDictionaryItem>();
        }

        private static void ReadAccessRights(IEnumerable<AccessDictionaryItem> items)
        {            
            foreach (AccessDictionaryItem item in items)
            {
                if (String.IsNullOrEmpty(item.Key))
                {
                    continue;
                }
                if (_accessRights.ContainsKey(item.Key))
                {
                    System.Console.Out.WriteLine("ERROR: Key duplicate in rights: {0}", item.Key);
                }
                else
                {
                    _accessRights.Add(item.Key, item);
                }                
            }

        }


        private static void ReadCheckableFromBins(Assembly assembly)
        {
            foreach (Type type in assembly.GetTypes().Where(t => t.GetCustomAttribute(typeof(CheckAccess)) != null))
            {
                string key = GetAccessKey(type);
                AddCheckable(key, type); 
            }
            foreach (Type type in assembly.GetTypes())
            {
                foreach (MethodInfo methodInfo in type.GetMethods().Where(t => t.GetCustomAttribute(typeof(CheckAccess)) != null))
                {
                    string key = GetAccessKey(methodInfo);
                    AddCheckable(key, methodInfo);
                }
            }
        }
        
        private static void AddCheckable(string key, MemberInfo member)
        {
            if (_checkables.ContainsKey(key))
            {

                _checkables[key].Add(member);
            }
            else
            {
                _checkables.Add(key,  new List<MemberInfo>() { member });
            }
        }

        private static void Correlate()
        {
            _memberRights.Clear();
            foreach (KeyValuePair<string, List<MemberInfo>> keyValuePair in _checkables)
            {
                foreach (MemberInfo member in keyValuePair.Value)
                {
                    _memberRights.Add(member,
                        _accessRights.ContainsKey(keyValuePair.Key) ? _accessRights[keyValuePair.Key] : null);
                }
                
            }            
        }
        
        public static void DoAction(MyAction func, MyActionParams prms = null)
        {
            if (GetIsAccessible(func))
            {
                func.Invoke(prms);
            }

        }

        public static bool GetIsAccessible(MyAction action)
        {
            CheckParam(action, "action");
            /*
            return GetIsAccessible(GetAccessKey(action.Method));
            */
            return GetIsAccessible(action.Method);
        }

        public static bool GetIsAccessible(Type type)
        {
            
            CheckParam(type, "type");
            /*
            return GetIsAccessible(GetAccessKey(type));
            */
            return GetIsAccessible((MemberInfo)type);
        }

        private static bool GetIsAccessible(MemberInfo member)
        {
            bool result = !_memberRights.ContainsKey(member) || GetIsAccessible(_memberRights[member]);
            System.Console.Out.WriteLine(String.Format("GetIsAccessible: {0}: {1}", member, result));
            return result;
        }

        private static bool GetIsAccessible(AccessDictionaryItem accessItem)
        {
            return accessItem != null && accessItem.IsAccessible;
        }

        /*
        private static bool GetIsAccessible(string key)
        {
            bool result;
            if (String.IsNullOrEmpty(key))
            {
                result = true;
                //only for tests
                key = "Empty key";
            }
            else
            {
                result = _accessRights.ContainsKey(key) && _accessRights[key].IsAccessible;
            }
            //only for tests
            System.Console.Out.WriteLine(String.Format("GetIsAccessible: {0}: {1}", key, result));
            return result;
        }
        */


        private static string GetAccessKey(Type type)
        {
            return GetAccessKey(type, info => type.FullName);
        }

        private static string GetAccessKey(MethodInfo method)
        {
            return GetAccessKey(method, info =>
            {
                Type declaringType = info.DeclaringType;
                CheckParam(declaringType, "declaringType");
                return String.Format("{0}.{1}.{2}", declaringType.Namespace, declaringType.Name, info.Name);
            });
        }


        /// <summary>
        /// Returns access key for member
        /// </summary>
        /// <param name="member"></param>
        /// <returns>null or "" if check is not required</returns>
        static private string GetAccessKey(MemberInfo member, Func<MemberInfo, string> keyGenerator)
        {
            CheckParam(member, "member");
            var attribute = (CheckAccess)member.GetCustomAttribute(typeof(CheckAccess));
            if (attribute == null)
            {
                return null;
            }
            string key = attribute.AccessKey;
            if (String.IsNullOrEmpty(key))
            {
                key = keyGenerator(member);
            }
            return key;

        }

        private static void CheckParam(Object obj, string title)
        {
            if (obj == null)
            {
                throw new Exception(String.Format("{0} is null", title));
            }

        }

        private static void SwapAccessRights()
        {
            foreach (KeyValuePair<string, AccessDictionaryItem> keyValuePair in _accessRights)
            {
                System.Console.Out.WriteLine(String.Format("{0} - IsAccessible: {1}", keyValuePair.Key, keyValuePair.Value.IsAccessible));
            }
        }

        private static void SwapCheckableFromBins()
        {
            foreach (KeyValuePair<string, List<MemberInfo>> keyValuePair in _checkables)
            {
                foreach (MemberInfo member in keyValuePair.Value)
                {
                    System.Console.Out.WriteLine(String.Format("Key: {0} Member: {1}", keyValuePair.Key, member));
                }
                
            }
        }

        private static void SwapMemberRights()
        {
            foreach (KeyValuePair<MemberInfo, AccessDictionaryItem> keyValuePair in _memberRights)
            {
                System.Console.Out.WriteLine(String.Format("Member: {0} IsAccessible: {1}", keyValuePair.Key, GetIsAccessible(keyValuePair.Value)));
            }
        }

        public static void Swap()
        {
            Program.WriteBlockHeader("Access Rights");
            SwapAccessRights();
            Program.WriteBlockHeader("Checkable From Bins");
            SwapCheckableFromBins();
            Program.WriteBlockHeader("Correlated Rights and Checkabels");
            SwapMemberRights();
        }

        public static IEnumerable<string> GetAccessKeys()
        {
            return _checkables.Keys;
        }

        public static void InitRights(List<AccessDictionaryItem> accessDictionaryItems)
        {
            _checkables.Clear();
            _accessRights.Clear();
            _memberRights.Clear();
            
            ReadAccessRights(accessDictionaryItems);            
            ReadCheckableFromBins(Assembly.GetExecutingAssembly());                       
            Correlate();
            
        }
    }

    public class AccessDictionaryItem
    {
        public string Key;
        public bool IsAccessible;
    }


    public class NonCheckableType
    {
    }

    [CheckAccess]
    public class NonAccessibleType
    {
    }

    [CheckAccess]
    public class AccessibleType
    {

        static public void NonCheckableAction(MyActionParams prms)
        {
            //check type is correct. for ex. MyActionParamsSample
            System.Console.Out.WriteLine("NonCheckableAction executed");
        }

        [CheckAccess]
        static public void AccesibleAction(MyActionParams prms)
        {
            //check type is correct. for ex. MyActionParamsSample
            System.Console.Out.WriteLine("AccesibleAction executed");
        }

        [CheckAccess]
        public static void NonAccesibleAction(MyActionParams prms)
        {
            //check type is correct. for ex. MyActionParamsSample
            System.Console.Out.WriteLine("NonAccesibleAction executed");
        }

        [CheckAccess("AccesibleActionWithAlias")]
        static public void AccesibleActionWithAlias(MyActionParams prms)
        {
            //check type is correct. for ex. MyActionParamsSample
            System.Console.Out.WriteLine("AccesibleActionWithAlias executed");
        }

        [CheckAccess("AccesibleActionWithAlias")]
        static public void AccesibleActionWithAliasDuplicate(MyActionParams prms)
        {
            //check type is correct. for ex. MyActionParamsSample
            System.Console.Out.WriteLine("AccesibleActionWithAliasDuplicate executed");
        }

        [CheckAccess("NonAccesibleActionWithAlias")]
        public static void NonAccesibleActionWithAlias(MyActionParams prms)
        {
            //check type is correct. for ex. MyActionParamsSample
            System.Console.Out.WriteLine("NonAccesibleActionWithAlias executed");
        }

    }

    [CheckAccess("AccessibleTypeWithAlias")]
    public class AccessibleTypeWithAlias
    {
    }

    //error
    [CheckAccess("AccessibleTypeWithAlias")]
    public class AccessibleTypeWithAliasDuplicate
    {
    }

    [CheckAccess("NonAccessibleTypeWithAlias")]
    public class NonAccessibleTypeWithAlias
    {
    }

    //error
    [CheckAccess("NonAccessibleTypeWithAlias")]
    public class NonAccessibleTypeWithAliasDuplicate
    {
    }
    class Program
    {
        static void Main(string[] args)
        {
            WriteBlockHeader("Read settings");
            AccessUtils.InitRights(new List<AccessDictionaryItem>()
                {
                    new AccessDictionaryItem() { IsAccessible = true, Key = "MethodOf.AccessibleType"},

                    new AccessDictionaryItem() { IsAccessible = true, Key = "MethodOf.AccessibleType.AccesibleAction"},
                    new AccessDictionaryItem() { IsAccessible = true, Key = "AccesibleActionWithAlias"},
                    new AccessDictionaryItem() { IsAccessible = true, Key = "AccessibleTypeWithAlias"},

                    new AccessDictionaryItem() { IsAccessible = false, Key = "NonAccesibleActionWithAlias"},

                    //error
                    new AccessDictionaryItem() { IsAccessible = false, Key = "NonAccesibleActionWithAlias"}
                });
            AccessUtils.Swap();

            WriteBlockHeader("All Keys");
            foreach (string accessKey in AccessUtils.GetAccessKeys())
            {
                System.Console.WriteLine(accessKey);
            }

            WriteBlockHeader("Check");            
            AccessUtils.GetIsAccessible(typeof(NonCheckableType));
            AccessUtils.GetIsAccessible(typeof(AccessibleType));
            AccessUtils.GetIsAccessible(typeof(AccessibleTypeWithAliasDuplicate));
            AccessUtils.GetIsAccessible(typeof(NonAccessibleType));
            

            AccessUtils.GetIsAccessible(AccessibleType.NonCheckableAction);
            AccessUtils.GetIsAccessible(AccessibleType.AccesibleAction);
            AccessUtils.GetIsAccessible(AccessibleType.AccesibleActionWithAlias);
            AccessUtils.GetIsAccessible(AccessibleType.AccesibleActionWithAliasDuplicate);

            AccessUtils.GetIsAccessible(AccessibleType.NonAccesibleAction);
            AccessUtils.GetIsAccessible(AccessibleType.NonAccesibleActionWithAlias);            
            
            WriteBlockHeader("Execute");
            AccessUtils.DoAction(AccessibleType.NonCheckableAction);
            AccessUtils.DoAction(AccessibleType.AccesibleAction);
            AccessUtils.DoAction(AccessibleType.AccesibleActionWithAlias);
            AccessUtils.DoAction(AccessibleType.AccesibleActionWithAliasDuplicate);

            AccessUtils.DoAction(AccessibleType.NonAccesibleAction);
            AccessUtils.DoAction(AccessibleType.NonAccesibleActionWithAlias);
            

            Console.In.ReadLine();
        }

        internal static void WriteBlockHeader(string block)
        {
            System.Console.Out.WriteLine();
            System.Console.Out.WriteLine(block);
            System.Console.Out.WriteLine("---------------");
            System.Console.Out.WriteLine();
        }
    }
}
