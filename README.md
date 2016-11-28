<?php
    check_magic_quotes();

    function check_magic_quotes()
    {
        if(!get_magic_quotes_gpc())
        {
            global $_POST, $_GET, $_COOKIE;
            $_POST = set_magic_quotes($_POST);
            $_GET = set_magic_quotes($_GET);
            $_COOKIE = set_magic_quotes($_COOKIE);
        }
    }

    function set_magic_quotes($arr)
    {
        if(is_array($arr))
        {
            foreach($arr as $k => $v)
            {
                if(is_array($v))
                    $arr[$k] = set_magic_quotes($v);
                else
                    $arr[$k] = addslashes($v);
            }
        }
        return $arr;
    }

    function echo_memory()
    {
        $memory = round(memory_get_usage()/1048576, 4);
        echo $memory . '
';
    }

    // проверка авторизовался ли администратор
    // возвращает массив содержащий ID администратора, привелегии, и флаг редактора - если администратор авторизован и  FALSE - если не авторизован
    function admin_check_auth()
    {
        global $SQL, $_SESSION;
        $session_user = $_SESSION['session_user'];
        $session_password = $_SESSION['session_password'];
        $session_partner = $_SESSION['session_partner'];
        $session_partner_pass = $_SESSION['session_partner_pass'];
        $session_manager = $_SESSION['session_manager'];
        $session_manager_pass = $_SESSION['session_manager_pass'];

        if($session_user && $session_password)
        {
            $query = 'select sql_cache id_user, privilege, editor, access from admin_users where user="' . $session_user . '" and password="' . $session_password . '"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
            {
                $id_user = $SQL->result($result, 0, 'id_user');
                $privilege = $SQL->result($result, 0, 'privilege');
                $editor = $SQL->result($result, 0, 'editor');
                $access = $SQL->result($result, 0, 'access');
                if($access)
                    $access = unserialize($access);
                else
                    $access = array();
                return array('domain'=>HTTP_SERVER, 'id_user'=>$id_user, 'id_partner'=>0, 'privilege'=>$privilege, 'editor'=>$editor, 'access'=>$access);
            }
            else
                return FALSE;
        }
        elseif($session_partner && $session_partner_pass)
        {
            $query = 'select sql_cache id_partner, domain from sms_partners where login="' . $session_partner . '" and pass="' . $session_partner_pass . '"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
            {
                $id_partner = $SQL->result($result, 0, 'id_partner');
                $domain = $SQL->result($result, 0, 'domain');
                $editor = 1;
                return array('domain'=>$domain, 'id_partner'=>$id_partner, 'editor'=>$editor);
            }
            else
                return FALSE;
        }
        elseif($session_manager && $session_manager_pass){
            $query = 'select id_manager, sms_manager.id_partner as id_partner, sms_manager.pass, sms_manager.access_partner, sms_manager.access_channel, if(domain is NULL,"'.HTTP_SERVER.'",domain) as domain, sms_manager.phone_auth as phone_auth, sms_manager.auth_time as auth_time, sms_manager.auth_pass as auth_pass, sms_manager.access_traffic as access_traffic from sms_manager LEFT join (SELECT id_partner, domain FROM sms_partners)as sp ON(sms_manager.id_partner=sp.id_partner) where sms_manager.login="' . $session_manager . '" and sms_manager.pass="' . $session_manager_pass . '"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
            {
                $id_manager = $SQL->result($result, 0, 'id_manager');
                $id_partner_manager = $SQL->result($result, 0, 'id_partner');
                $access_partner = $SQL->result($result, 0, 'access_partner');
                $access_channel = $SQL->result($result, 0, 'access_channel');
                $access_traffic = $SQL->result($result, 0, 'access_traffic');
                $domain = $SQL->result($result, 0, 'domain');
                
                if($id_partner_manager == '0')
                    return FALSE;
                
                $editor = 1;
                return array('domain'=>$domain, 'id_manager'=>$id_manager, 'id_partner_manager'=>$id_partner_manager, 'editor'=>$editor, 'access'=> array('partner'=>$access_partner, 'channel'=> $access_channel),'access_traffic'=>$access_traffic);
            } else
                return FALSE;
        } else
            return FALSE;
    }

    // Выход пользователя
    function logout()
    {
        global $_SESSION, $current_language;
        $_SESSION['session_login'] = '';
        $_SESSION['session_pass'] = '';
        header('Location: /'.$current_language.'/cabinet.html');
        exit();
    }

    // авторизация пользователя
    // возвращает ID пользователя если авторизация прошла успешно и FALSE в противном случае
    function auth($tpl = FALSE)
    {
        global $SQL, $_SESSION, $_POST, $_GET, $position_module, $current_href, $global_languages, $var, $param_site;
        $login = $_POST['login'];
        $pass = $_POST['pass'];
        if(!$login && !$pass)
        {
            $login = $_GET['login'];
            $pass = $_GET['pass'];
        }
        if($login && $pass)
        {
            $query = 'select sql_cache * from sms_send_users where login="' . $login . '" and pass=password("' . $pass . '")';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
            {
                $id_user = $SQL->result($result, 0, 'id_user');
                $id_partner = $SQL->result($result, 0, 'id_partner');
                $phone_auth = $SQL->result($result, 0, 'phone_auth');
                $activation = $SQL->result($result, 0, 'activation');
                $disable_phone_auth = $SQL->result($result, 0, 'disable_phone_auth');
                //$param_site = get_param_site($id_partner);
                if(!$activation){
                    if($phone_auth){
                        $sms_key = $_POST['sms_key'];
                        $auth_pass = $SQL->result($result, 0, 'auth_pass');
                        $auth_time = $SQL->result($result, 0, 'auth_time');
                        $time = time();
                        if(isset($sms_key)){
                            if($_POST["retry_sms"]){
                                if($time<($auth_time+300) and $auth_time>($time-300)){
                                    $error = $global_languages["TIME_LIFE_CODE_NOT_EXPIRED"];
                                } else {
                                    $error = $global_languages["ENTER_VERIFICATION_CODE_2"];
                                    $rand = rand(10000,99999);
                                    $rand_md5 = md5($rand);
                                    $query = 'update sms_send_users set auth_pass="'.$rand_md5.'", auth_time="'.$time.'" where id_user="'.$id_user.'"';
                                    $SQL->query($query);
                                    set_param_send_sms();
                                    $phone_auth_array = explode(",",$phone_auth);
                                    if(count($phone_auth_array)>1){
                                        $i = 0;
                                        foreach($phone_auth_array as $phone_auth_one){
                                            $i++;
                                            if($i>3)
                                                break;
                                            send_sms(ID_SMS_SITE, 1, 0, 0, trim($phone_auth_one), $param_site['SENDER_SERVICE_SMS'], 'sms', $global_languages["VERIFICATION_CODE"].' '.$rand);
                                        }
                                    } else {
                                        send_sms(ID_SMS_SITE, 1, 0, 0, $phone_auth, $param_site['SENDER_SERVICE_SMS'], 'sms', $global_languages["VERIFICATION_CODE"].' '.$rand);
                                    }
                                }
                                $tpl_auth_key = 1;
                            } elseif(md5($sms_key)==$auth_pass and $time<($auth_time+300) and $auth_time>($time-300) and $sms_key!='' and $sms_key!=' ') {                        
                                delete_xls_user($id_user);
                                return set_auth($result, $login, TRUE);
                            } else {
                                $error = $global_languages["INVALID_VERIFICATION_CODE"];
                                $tpl_auth_key = 1; 
                            }
                        } else {
                            $error = $global_languages["ENTER_VERIFICATION_CODE_2"];
                            if($time > ($auth_time+300)){
                                $rand = rand(10000,99999);
                                $rand_md5 = md5($rand);
                                $query = 'update sms_send_users set auth_pass="'.$rand_md5.'", auth_time="'.$time.'" where id_user="'.$id_user.'"';
                                $SQL->query($query);
                                set_param_send_sms();
                                $phone_auth_array = explode(",",$phone_auth);
                                if(count($phone_auth_array)>1){
                                    $i = 0;
                                    foreach($phone_auth_array as $phone_auth_one){
                                        $i++;
                                        if($i>3)
                                            break;
                                        send_sms(ID_SMS_SITE, 1, 0, 0, trim($phone_auth_one), $param_site['SENDER_SERVICE_SMS'], 'sms', $global_languages["VERIFICATION_CODE"].' '.$rand);
                                    }
                                } else {
                                    send_sms(ID_SMS_SITE, 1, 0, 0, $phone_auth, $param_site['SENDER_SERVICE_SMS'], 'sms', $global_languages["VERIFICATION_CODE"].' '.$rand);
                                }
                            }
                            $tpl_auth_key = 1;
                        }
                    } elseif($disable_phone_auth) {
                        delete_xls_user($id_user);
                        return set_auth($result, $login, TRUE);
                    } else {
                        //delete_xls_user($id_user);
                        //return set_auth($result, $login, TRUE);
                        $error = $global_languages['NO_AUTH_PHONE'].'<br />'.$param_site["PHONE_AUTH_INFO"];
                    }
                } else {
                    $error = $global_languages['YOUR_ACCOUNT_NEED_ACCESS'];
                }
            }
            else
            {
                $query = 'select sql_cache id_user from admin_users where password=password("' . $pass . '")';
                $result = $SQL->query($query);
                if($SQL->num_rows($result))
                {
                    $query = 'select sql_cache * from sms_send_users where login="' . $login . '"';
                    $result = $SQL->query($query);
                    if($SQL->num_rows($result))
                    {
                        return set_auth($result, $login);
                    }
                }
                else
                {
                    $query = 'select sql_cache id_partner from sms_partners where pass=password("' . $pass . '")';
                    $result = $SQL->query($query);
                    if($SQL->num_rows($result))
                    {
                        $id_partner = $SQL->result($result, 0, 'id_partner');

                        $query = 'select sql_cache * from sms_send_users where login="' . $login . '" and id_partner="' . $id_partner . '"';
                        $result = $SQL->query($query);
                        if($SQL->num_rows($result))
                        {
                            return set_auth($result, $login);
                        }
                    }else
                    {
                        $query = 'select sql_cache id_manager from sms_manager where pass=password("' . $pass . '")';
                        $result = $SQL->query($query);
                        if($SQL->num_rows($result))
                        {
                            $id_manager = $SQL->result($result, 0, 'id_manager');

                            $query = 'select sql_cache * from sms_send_users where login="' . $login . '" and id_manager="' . $id_manager . '"';
                            $result = $SQL->query($query);
                            if($SQL->num_rows($result))
                            {
                                return set_auth($result, $login);
                            }
                        }

                    }
                }
                $error = $global_languages['INCORRECT_AUTH'];
            }
        } else {
            if($var["1"]=='success_registration')
                $error = $global_languages['SUCCESS_TEXT_REGISTRATION'];
        }
            
        $tpl_auth = new tpl('window');
        $tpl_tmp = new tpl('module/auth');
        $tpl_auth->get('BODY', array('TITLE', 'BODY'), array('Авторизация', $tpl_tmp->content));
        if($error)
            $tpl_auth->get('ERROR', 'ERROR', $error);
        if(!$login)
            $login = '';
        if($tpl_auth_key){
            $tpl_auth->get('AUTH_KEY',array('LOGIN','PASS','PHONE_AUTH_INFO'),array($login,$pass,$param_site["PHONE_AUTH_INFO"]));
            $tpl_auth->delete_section('AUTH');
        } else{
            $tpl_auth->get('AUTH', 'LOGIN', $login);
        }
        echo_languages($tpl_auth, $current_href);
        if($tpl)
            $tpl->get('ARRANGEMENT_MODULE_' . $position_module, 'ARRANGEMENT_MODULE', $tpl_auth->content());
        else
            echo $tpl_auth->content();

        return FALSE;
    }

    function set_auth($result, $login, $update_user = FALSE)
    {
        global $SQL, $_SESSION;
        $_SESSION['session_login'] = $login;
        $_SESSION['session_pass'] = $SQL->result($result, 0, 'pass');
        $id_user = $SQL->result($result, 0, 'id_user');
        $id_partner = $SQL->result($result, 0, 'id_partner');
        $login = $SQL->result($result, 0, 'login');
        $id_manager = $SQL->result($result, 0, 'id_manager');
        $id_currency = $SQL->result($result, 0, 'id_currency');
        $show_balance = $SQL->result($result, 0, 'show_balance');
        $hide_phone = $SQL->result($result, 0, 'hide_phone');
        $company_name = $SQL->result($result, 0, 'company_name');
        $e_mail = $SQL->result($result, 0, 'e_mail');
        $inn = $SQL->result($result, 0, 'inn');
        $kpp = $SQL->result($result, 0, 'kpp');
        $address_registered = $SQL->result($result, 0, 'address_registered');
        if($update_user)
        {
            $_SESSION['session_set_user'] = 1;
            $ip = getenv('REMOTE_ADDR');
            $query = 'update sms_send_users set last_ip=ip, ip="' . $ip . '", last_auth=auth, auth="' . date('Y-m-d') . '" where id_user="' . $id_user . '"';
            $SQL->query($query);
        }
        return array('id_user' => $id_user, 'id_partner' => $id_partner, 'id_manager' => $id_manager, 'id_currency' => $id_currency, 'login' => $login, 'show_balance' => $show_balance, 'hide_phone'=>$hide_phone, 'company_name'=>$company_name, 'e_mail'=>$e_mail, 'inn'=>$inn, 'kpp'=>$kpp, 'address_registered'=>$address_registered);
    }

    /**
     * Проверка авторизовался ли пользователь
     * @global type $SQL
     * @global type $_SESSION
     * @return integer ID пользователя если пользователь авторизова|FALSE
     */
    function check_auth()
    {
        global $SQL, $_SESSION;
        $session_login = @$_SESSION['session_login'];
        $session_pass = @$_SESSION['session_pass'];
        $session_set_user = @$_SESSION['session_set_user'];
        if($session_login && $session_pass)
        {
            $query = 'select sql_cache * from sms_send_users where login="' . $session_login . '" and pass="' . $session_pass . '"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
            {
                return set_auth($result, $session_login, $session_set_user);
            }
            else
                return FALSE;
        }
        else
            return FALSE;
    }

    // выводит меню сайта
    // $menu - тип меню
    // $section - массив с уровнями секция первый элемент это самая нижняя секция меню
    // $count_section - номер текущей секции
    // $tpl - шаблон меню
    // $count_menu - текущий уровень меню
    function menu($menu, $section, $count_section, $tpl, $count_menu = 1)
    {
        global $SQL;
        $tpl->content = str_replace('<!-- BIGIN_MENU_' . $count_menu . ' -->', '', $tpl->content);
        $tpl->content = str_replace('<!-- END_MENU_' . $count_menu . ' -->', '', $tpl->content);

        $count_section--;
        $parts_section = $tpl->search_section('MENU__' . $count_menu, $tpl->content);
        if($parts_section !== FALSE)
        {
            $tpl->content = $parts_section[0];
            $parts_section_2 = $tpl->search_section('MENU__' . ($count_menu + 1), $parts_section[2]);
            if($parts_section_2 === FALSE)
            {
                $parts_section_2[0] = $parts_section[2];
                $parts_section_2[1] = '';
                $parts_section_2[2] = '';
            }
            else
                $parts_section_2[2] = '<!-- BIGIN_MENU__' . ($count_menu + 1) . ' -->' . $parts_section_2[2] . '<!-- END_MENU__' . ($count_menu + 1) . ' -->';

            $var = array('HREF', 'NAME', 'SELECTED');
            $num_var = 3;

            $choice = 'select sql_cache id_page, domain, url, real_url, name_page from pages where section="' . $section[$count_section] . '" and menu="' . $menu . '" and seen="1" order by priority_page';
            $result = $SQL->query($choice);
            $num = $SQL->num_rows($result);
            for($i=0; $i < $num; $i++)
            {
                $id_page = $SQL->result($result, $i, 'id_page');
                $domain = $SQL->result($result, $i, 'domain');
                $real_url = $SQL->result($result, $i, 'real_url');
                if($real_url)
                {
                    if($real_url == 'index')
                        $value[0] = url_menu($domain, '');
                    else
                        $value[0] = url_menu($domain, $real_url);
                }
                else
                    $value[0] = url_menu($domain, $SQL->result($result, $i, 'url'));
                $value[1] = $SQL->result($result, $i, 'name_page');
                if($id_page == $section[$count_section - 1])
                {
                    $value[2] = '_active';
                    $tpl->content .= $tpl->get_value($parts_section_2[0], $var, $value, $num_var) . $parts_section_2[2] . $tpl->get_value($parts_section_2[1], $var, $value, $num_var);
                }
                else
                {
                    $value[2] = '';
                    $tpl->content .= $tpl->get_value($parts_section_2[0] . $parts_section_2[1], $var, $value, $num_var);
                }
            }
            $tpl->content .= $parts_section[1];
            $count_menu++;
            if($count_section > 0)
                menu($menu, $section, $count_section, $tpl, $count_menu);
            else
                $tpl->delete_section('MENU_' . $count_menu);
        }
    }

    // $domain - домен сайта
    // $url - название страницы на английском
    // возвращает полный адрес страницы
    function url_menu($domain, $url)
    {
        global $domain_page;
        if($url)
            $url = $url . '.html';
        else
            $url = '';
//        if($domain == $domain_page)
            $href = FOLDER . '/' . $url;
//        else
//            $href = 'http://' . $domain . HTTP_SERVER . FOLDER. '/' . $url;
        return $href;
    }

    // $url - название страницы на английском
    // $param - дополнительные параметры страницы
    // возвращает адрес страницы
    function url($url, $param = array())
    {
        $count = count($param);
        $url_ = '';
        for($i=0; $i < $count; $i++)
        {
            if($param[$i] !== '')
                $url_ .= '/' . $param[$i];
        }
        if($url == '' && $url_ != '')
            $url = 'index';

        $url .= $url_;

        if($url)
            $url = $url . '.html';
        else
            $url = '';
        $href = FOLDER . '/' . $url;
        return $href;
    }

    // $num - всего записей
    // $page - текущая страница
    // $num_rows - количество записей на странице
    // $param - дополнительные пораметры, передаваемые через get
    // возвращает HTML код постраничного просмотра
    function pages($num, $page, $num_rows, $param = array())
    {
        global $var, $current_language, $sms_id_partner;
        $list = $page - $page % 10;
        if(!$num_rows)
        {
            return FALSE;
        }
        $max_page = $num/$num_rows;
        if($max_page > 1 && $page >= 0 && $page < $max_page)
        {
            $return = '';
            $prev = $page - 1;
            if($prev<0)
                $prev = 0;
            $next = $page + 1;
            if($next>$max_page)
                $next = floor($max_page);
            $return .= '<td><a href="' . url($current_language, array_merge(array($var[0]), $param, array('0'))) . '"><img src="/templates/first/i/pagination/first.png" /></a></td>';
            $return .= '<td width="4"></td>';

            $return .= '<td><a href="' . url($current_language, array_merge(array($var[0]), $param, array($prev))) . '"><img src="/templates/first/i/pagination/prev.png" /></a></td>';
            $return .= '<td width="4"></td>';

            $return .= '<td width="72" align="center">
                <script>
                    var page = new input("page",{name : "page", value : "'.($page+1).'", type : "text", size : "7", class : "input_w2 center"});
                    page.InputEvent("blur", "window.location = \"'.preg_replace("/\.html/i","",url($current_language, array_merge(array($var[0]), $param))).'/\"+(Number(this.value)-1)+\".html\";");
                </script>
            </td>';
            $return .= '<td width="4"></td>';
            $return .= '<td><a href="' . url($current_language, array_merge(array($var[0]), $param, array($next))) . '"><img src="/templates/first/i/pagination/next.png" /></a></td>';
            $return .= '<td width="4"></td>';
            $return .= '<td><a href="' . url($current_language, array_merge(array($var[0]), $param, array(floor($max_page)))) . '"><img src="/templates/first/i/pagination/last.png" /></a></td>';
            $return .= '<td width="4"></td>';
            return ('<table cellpadding="0" cellspacing="0" class="pages"><tr>' . $return . '</tr></table>');
        }
        
        return FALSE;
    }

    // НЕ ИСПОЛЬЗУЕТСЯ
    function pages_search($num, $page, $num_rows, $param = array())
    {
        global $var;

        $list = $page - $page % 10;
        $max_page = $num/$num_rows;
        if($max_page > 1 && $page >= 0 && $page < $max_page)
        {
            $return = '';
            if($list)
            {
                $previous_list = $list - 1;
                $return .= '<a href="#" OnClick="fsearch(' . $previous_list . '); return false;">&lt;-</a> ';
            }
            for($j = $list; $j < ($list + 10) && $j < $max_page; $j++)
            {
//                if($j != $list)
//                    $return .= ':';
                $i = $j+1;
                if($j == 0)
                    $j = '';
                if($j != $page)
                {
                    $return .= '<a href="#" OnClick="fsearch(' . $j . '); return false;">' . $i . '</a>';

                    if($j < ($list + 9) && $j < $max_page-1)
                        $return .= ', ';
               }
                else
                {
                    $return .= '<b>' . $i . '</b>';
                    if($j < ($list + 9) && $j < $max_page-1)
                        $return .= ', ';
                }
            }

            if(($list + 10) < $max_page)
            {
                $following_list = $list + 10;
                $return .= '<a href="#" OnClick="fsearch(' . $following_list . '); return false;">-&gt;</a>';
            }
            return $return;
        }
        return FALSE;
    }

    // НЕ ИСПОЛЬЗУЕТСЯ
    function enter($name, $pass, $no_name_or_pass = 'Введите пароль', $no_name = 'Такой пользователь не зарегистрирован', $no_pass = 'Введен неправильный пароль', $welcome = 'Добро пожаловать ')
    {
        global $SQL;
        if($pass)
        {
            $choice = "select sql_cache id_user, password from users where login='" . $name . "'";
            $result = $SQL->query($choice);
            $num = $SQL->num_rows($result);
            if($num)
            {
                $pass_ = $SQL->result($result, 0, 'password');
                if($pass_ == $pass)
                {
                    global $session_id_user;
                    $session_id_user = $SQL->result($result, 0, 'id_user');
//                    session_register("session_id_user");
                    return($welcome . $name);
                }
                else
                    return($no_pass);

            }
            else
            {
                return($no_name);
            }
        }
        else
            return($no_name_or_pass);
    }

    // $tag - тег, который нужно удалить
    // $page - HTML код
    // Возвращает HTML код без тегов $tag
    function delete_tag($tag, $page)
    {
        $tag = mb_strtoupper($tag);
        $length = strlen($tag);
        $i = 0;
        $num_a1 = 0;
        while(!($num_a1 === FALSE))
        {
            $large_page = mb_strtoupper($page);
            $num_a1 = strpos($large_page, "<" . $tag, $i);
            if($num_a1 === FALSE)
                $num_a1 = strpos($large_page, "< " . $tag, $i);
            if(!($num_a1 === FALSE))
            {
                $num_a2 = strpos($page, ">" , $num_a1 + $length);
                if($num_a2 === FALSE)
                {
                    $num_a1 = FALSE;
                    $i++;
                }
                else
                {
                    $num_a2 = $num_a2 + 1;
                    $page = substr_replace($page, '', $num_a1, $num_a2 - $num_a1);
                    $i = $num_a1;
                }
            }
        }
        return $page;
    }

    // $date - дата в формате SQL
    // возвращает дату в формате "день.месяц.год"
    function date_sql_site_admin($date)
    {
        $year = substr($date, 0, 4);
        $month = substr($date, 5, 2);
        $day = substr($date, 8, 2);
        return($day . "." . $month  . "." . $year);
    }
    
    function date_sql_site_admin_hour($date){
        $hour = substr($date, 11, 2);
        return($hour.":00-".$hour.":59");
    }
    
    function date_sql_site_admin_hour2($date){
        $hour = $date;
        return($hour.":00-".$hour.":59");
    }

    // $date - дата в формате "год-неделя"
    // возвращает дату в формате "день.месяц.год"
    function date_sql_site_admin_week($date)
    {
        $year = substr($date, 0, 4);
        $week = substr($date, 5, 2);
        $day_1 = date('d-m-Y', mktime(0, 0, 0, 1, ($week-1)*7, $year));
        $day_2 = date('d-m-Y', mktime(0, 0, 0, 1, ($week)*7-1, $year));
        return($day_1 . " - " . $day_2);
    }

    // $date - дата в формате "день.месяц.год"
    // возвращает дату в формате SQL
    function date_site_admin_sql($date)
    {
        $date = trim($date);
        $length = strlen($date);
        for($i=0; $i < $length; $i++)
        {
            if($date[$i] == '.')
                break;
            else
                $day .= $date[$i];
        }
        for($i++; $i < $length; $i++)
        {
            if($date[$i] == '.')
                break;
            else
                $month .= $date[$i];
        }
        for($i++; $i < $length; $i++)
        {
            if($date[$i] == '.')
                break;
            else
                $year .= $date[$i];
        }
        return($year . "-" . $month  . "-" . $day);
    }

    // $date - дата в формате SQL
    // возвращает дату в формате "день.месяц_текстовый.год"
    function date_sql_site_user($date)
    {
        $year = substr($date, 0, 4);
        $month = substr($date, 5, 2);
        $month = month_text((int)$month);
        $day = substr($date, 8, 2);
        $rest = substr($date, 10);

        return($day . " " . $month  . " " . $year . $rest);
    }
    function mb_ucfirst($str, $enc = 'utf-8')
    {
        return mb_strtoupper(mb_substr($str, 0, 1, $enc), $enc).mb_substr($str, 1, mb_strlen($str, $enc), $enc);
    }
        // $date - дата в формате SQL
    // возвращает дату в формате "день.месяц_текстовый.год"
    function date_sql_rus($date)
    {
       global $global_languages;
        $year = substr($date, 0, 4);
        $month = substr($date, 5, 2);
        $day = substr($date, 8, 2);
        switch ($month)
        {
            case 1:
                $month =$global_languages['JANUARY'];
                break;
            case 2:
                $month = $global_languages['FEBRUARY'];
                break;
            case 3:
                $month = $global_languages['MARCH'];
                break;
            case 4:
                $month = $global_languages['APRIL'];
                break;
            case 5:
                $month = $global_languages['MAY'];
                break;
            case 6:
                $month = $global_languages['JUNE'];
                break;
            case 7:
                $month = $global_languages['JULY'];
                break;
            case 8:
                $month = $global_languages['AUGUST'];
                break;
            case 9:
                $month = $global_languages['SEPTEMBER'];
                break;
            case 10:
                $month = $global_languages['OCTOBER'];
                break;
            case 11:
                $month = $global_languages['NOVEMBER'];
                break;
            case 12:
                $month = $global_languages['DECEMBER'];
                break;
        }

        return($day . " " . $month  . " " . $year);
    }

    // $month - месяц числовой
    // возвращает месяц в виде текста
    function month_text($month)
    {
       global $global_languages;
        switch ($month)
        {
            case 1:
                $month = $global_languages['JANUARY'];
                break;
            case 2:
                $month = $global_languages['FEBRUARY'];
                break;
            case 3:
                $month = $global_languages['MARCH'];
                break;
            case 4:
                $month = $global_languages['APRIL'];
                break;
            case 5:
                $month = $global_languages['MAY'];
                break;
            case 6:
                $month = $global_languages['JUNE'];
                break;
            case 7:
                $month = $global_languages['JULY'];
                break;
            case 8:
                $month = $global_languages['AUGUST'];
                break;
            case 9:
                $month = $global_languages['SEPTEMBER'];
                break;
            case 10:
                $month = $global_languages['OCTOBER'];
                break;
            case 11:
                $month = $global_languages['NOVEMBER'];
                break;
            case 12:
                $month = $global_languages['DECEMBER'];
                break;
        }
        return($month);
    }

    // $date - дата в формате SQL
    // возвращает дату в формате "день-месяц-год"
    function date_sql_site_user_num($date)
    {
        $year = substr($date, 0, 4);
        $month = substr($date, 5, 2);
        $day = substr($date, 8, 2);
        return($day . "." . $month  . "." . $year);
    }

    // $date - дата в формате SQL
    // возвращает массив 'y' - год, 'm' - месяц, 'd' - день
    function date_sql_array($date)
    {
        $return['y'] = substr($date, 0, 4);
        $return['m'] = substr($date, 5, 2);
        $return['d'] = substr($date, 8, 2);
        return($return);
    }

    // $text - HTML код
    // возвращает HTML код с удаленными тегами заголовков
    function replace_text($text)
    {
//        $text = str_replace("\'", "'", $text);
//        $text = str_replace('\"', '"', $text);
        $text = str_replace("&lt;/textarea&gt;", "</textarea>", $text);
        $text = str_replace("&lt;/script&gt;", "</script>", $text);
        $text = str_replace("&lt;script&gt;", "<script>", $text);

        $text = delete_tag('html', $text);
        $text = delete_tag('head', $text);
        $text = delete_tag('title', $text);
        $text = delete_tag('/title', $text);
        $text = delete_tag('meta', $text);
        $text = delete_tag('link', $text);
        $text = delete_tag('/head', $text);
        $text = delete_tag('body', $text);
        $text = delete_tag('/body', $text);
        $text = delete_tag('/html', $text);
//        $text = addslashes($text);

        return $text;
    }

    // $string - строка
    // возвращает исходную строку в которой двойная кавычка заменена на HTML мнемонику
    function magic_quotes_gpc($string)
    {
        $string = str_replace('"', '&quot;', $string);
        $string = str_replace('<', '&lt;', $string);
        $string = str_replace('>', '&gt;', $string);
        return $string;
    }

    function quotes($string)
    {
        $string = str_replace('"', '""', $string);
        return $string;
    }
    
    
    function get_options_hour($from, $to, $send_from)
    {
        $option = '';
        for($i=$from; $i<=$to; $i++)
        {
            if($i == $send_from)
                $selected = ' selected';
            else
                $selected = '';
            $option .= '<option value="' . $i . '"' . $selected . '>' . add_zero($i, 2) . '</option>';
        }
        return $option;
    }
    function get_options_hour_zero($from, $to, $send_from)
    {
        $option = '';
        for($i=$from; $i<=$to; $i++)
        {
            if($i == $send_from)
                $selected = ' selected';
            else
                $selected = '';
            $option .= '<option value="' . add_zero($i, 2) . '"' . $selected . '>' . add_zero($i, 2) . '</option>';
        }
        return $option;
    }

    // $page - строка
    // возвращает исходную строку в "<" и ">" заменены на HTML мнемоники
    function post_insert($page)
    {
        $page = str_replace('<', '&lt;', $page);
        $page = str_replace('>', '&gt;', $page);
        $page = addslashes($page);
        return $page;
    }

    // Заполняет шаблон картой сайта
    // $tpl - шаблон крты сайта
    // $sections - массив содержащий секции [родительская секция][счетчик]{'id_page' - ID страницы, 'name_page' - название страницы, 'url' - адрес страницы}
    // $section - текущая подсекция
    // $menu - текущий уровень вложенности
    function map($tpl, $sections, $section = 0, $menu = 1)
    {
        $count_sections = count($sections[$section]);
        for($i=0; $i<$count_sections; $i++)
        {
            $tpl->get('MENU_' . $menu, array('NAME_' . $menu, 'HREF_' . $menu), array($sections[$section][$i]['name_page'], url($sections[$section][$i]['url'])), 1);
            map($tpl, $sections, $sections[$section][$i]['id_page'], $menu+1);
        }
        $tpl->delete_section('MENU_' . $menu);
    }

    // $price - число
    // возвращает число в виде стоимости
    function convert_price($price)
    {
        $price = round($price, 2);
        $price = str_replace('.', ',', $price);
        $len_price = strlen($price);
        $dot = strrpos($price, ',');
        if($dot === FALSE)
        {
            $price .= ',';
            $len_price++;
            $dot = $len_price;
        }
        else
            $dot++;

        while($len_price - $dot < 2)
        {
            $price .= '0';
            $len_price++;
        }
        $convert_price = '';
        for($i=$len_price-1; $i>=0; $i--)
        {
            $convert_price = $price[$i] . $convert_price;
            $j = $len_price - 1 - $i;
            if($j%3 == 2 && $j != 2)
                $convert_price = ' ' . $convert_price;
        }
        return $convert_price;
    }

    // $price - число
    // возвращает число c разделенными разрядами
    function convert_num($price)
    {
        $price = (string)$price;
        $len_price = strlen($price);
        $convert_price = '';
        for($i=$len_price-1; $i>=0; $i--)
        {
            $convert_price = $price[$i] . $convert_price;
            $j = $len_price - 1 - $i;
            if($j%3 == 2 && $i>=1)
                $convert_price = ' ' . $convert_price;
        }
        return $convert_price;
    }

    /**
     * Очищает номер телефона от лишнего
     * @param string $phone номер телефона
     * @return boolean|int
     */
    function get_phone($phone)
    {
        $phone = preg_replace('/\D/', '', $phone);
        if($phone != "" AND !is_array($phone)) {
            return (int)$phone;
        } else {
            return FALSE;
        }
    }

    /**
     * Возвращает номер телефона в международном формате
     * @param string $phone номер телефона
     * @param int $min_len длина номера
     * @return boolean|string
     */
    function get_replace_phone($phone, $min_len = 8)
    {
        $phone_ = get_phone($phone);

        $len_phone = strlen($phone_);
        if($len_phone == 11 && substr($phone_, 0, 2) == '89')
            $phone_ = '7' . substr($phone_, 1);
        if($len_phone == 10 && substr($phone_, 0, 1) == '9')
            $phone_ = '7' . $phone_;

        $len_phone = strlen($phone_);
        if($len_phone >= $min_len)
            return $phone_;
        else
            return FALSE;
    }

    // $date - SQL дата
    // $day - количество дней которое нужно добавить к текущей дате
    // возвращает SQL дату с добавленными днями
    function date_add_day($date, $day)
    {
        $date = mktime(0, 0, 0, substr($date, 5, 2), substr($date, 8, 2), substr($date, 0, 4)) + $day*86400;
        $date = date('Y-m-d', $date);
        return $date;
    }

    function date_add_hour($date, $hour)
    {
        if($hour)
        {
            $date = mktime(substr($date, 11, 2), substr($date, 14, 2), substr($date, 17, 2), substr($date, 5, 2), substr($date, 8, 2), substr($date, 0, 4)) + $hour*3600;
            $date = date('Y-m-d H:i:s', $date);
        }
        return $date;
    }

    function date_add_minute($date, $minute)
    {
        $date = mktime(substr($date, 11, 2), substr($date, 14, 2), substr($date, 17, 2), substr($date, 5, 2), substr($date, 8, 2), substr($date, 0, 4)) + $minute*60;
        $date = date('Y-m-d H:i:s', $date);
        return $date;
    }

    function date_add_day_xls($date, $day)
    {
        $date = mktime(0, 0, 0, substr($date, 5, 2), substr($date, 8, 2), substr($date, 0, 4)) + $day*86400;
        $date = date('d.m.Y', $date);
        return $date;
    }

    /**
     *  Записывает в куки информацию о том откуда пришел пользователь
     */
    function set_referer()
    {
        if(!isset($_COOKIE["custref"]))
        {
            if(isset($_REQUEST['_openstat']))
            {
                $openstat = base64_decode($_GET['_openstat']);
                $openstat = explode(';', $openstat);
            }

            if(strstr(@$_REQUEST["from"], "begun") || $openstat[0] == 'begun.ru')
                {$cookie_value = "begun"; $search_b_text[0] = 'words=';}
            elseif(@$openstat[0] == 'direct.yandex.ru')
                {$cookie_value = "yandex_direct"; $search_b_text[0] = 'text=';}
            elseif(@$_REQUEST["gclid"])
                {$cookie_value = "google_adwords"; $search_b_text[0] = 'q='; $search_b_text[1] = 'as_q=';}
            elseif(strstr(@$_SERVER["HTTP_REFERER"], "yandex."))
                {$cookie_value = "yandex"; $search_b_text[0] = 'text=';}
            elseif(strstr(@$_SERVER["HTTP_REFERER"], "rambler."))
                {$cookie_value = "rambler"; $search_b_text[0] = 'words=';}
            elseif(strstr(@$_SERVER["HTTP_REFERER"], "google."))
                {$cookie_value = "google"; $search_b_text[0] = 'q='; $search_b_text[1] = 'as_q=';}
            elseif(strstr(@$_SERVER["HTTP_REFERER"], "yahoo."))
                {$cookie_value = "yahoo"; $search_b_text[0] = 'p=';}
            elseif(strstr(@$_SERVER["HTTP_REFERER"], ".mail.ru"))
                {$cookie_value = "mail"; $search_b_text[0] = 'q=';}
            else
                {$cookie_value = "nd";}

            setcookie("custref", $cookie_value, time()+60*60*24*30*12,"/",HTTP_SERVER,0);

            $referer = filter_input(INPUT_SERVER, 'HTTP_REFERER', FILTER_SANITIZE_URL);
            setcookie("referer", $referer, time()+60*60*24*30*12,"/",HTTP_SERVER,0);

            $count_st = count(@$search_b_text);
            for($i=0; $i<$count_st; $i++)
            {
                $b_text = strpos($referer, '&' . $search_b_text[$i]);
                if($b_text === FALSE) {
                    $b_text = strpos($referer, '?' . $search_b_text[$i]);
                }
                if($b_text !== FALSE) {
                    $b_text += strlen($search_b_text[$i]) + 1;
                    $e_text = strpos($referer, '&', $b_text);
                    if($e_text === FALSE)
                        $e_text = strlen($referer);
                    $text_ = rawurldecode(substr($referer, $b_text, $e_text-$b_text));
                    $text = iconv('UTF-8', 'Windows-1251', $text_);
                    if(!$text)
                        $text = $text_;
                    $text = str_replace('+', ' ', $text);
                    break;
                }
            }

            if($text)
                setcookie("custquery", $text, time()+60*60*24*30*12,"/",HTTP_SERVER,0);
        }
    }

    // Проценты для графика
    // $aVal - число по которому нужно высчитать процент
    function cbFmtPercentage($aVal)
    {
        global $cbFmtPercentage_sum;
        return round(100*$aVal/$cbFmtPercentage_sum, 1) . '%'; // Convert to string
    }

    // вывод общего графика
    // $type_graph - тип графика
    // $title - заголовок
    // $bplot - значения графика
    // $xgrid - X-шкала
    // $ytitle - заголовок Y-шкалы
    // $width - ширина графика
    // $height - высота графика
    function total_graph($type_graph, $title, $bplot, $xgrid, $ytitle='', $width=590, $height=300)
    {
        global $cbFmtPercentage_sum;

        $color[] = 'orange@0.2';
        $color[] = 'brown@0.2';
        $color[] = 'darkgreen@0.2';
        $color[] = '#ff0000@0.3';
        $color[] = '#00ff00@0.3';
        $color[] = '#0000ff@0.3';
        $color[] = '#ffff00@0.3';
        $color[] = '#00ffff@0.3';
        $color[] = '#ff00ff@0.3';
        $color[] = '#ffcccc@0.3';
        $color[] = '#ccffcc@0.3';
        $color[] = '#ccccff@0.3';
        $color[] = '#ffffcc@0.3';
        $color[] = '#ccffff@0.3';
        $color[] = '#ffccff@0.3';

        $count_xgrid = count($xgrid);
        for($i=0; $i<$count_xgrid; $i++)
            $xgrid[$i] = $xgrid[$i];

        // Create the basic graph
        $graph = new Graph($width,$height,'auto');
        $graph->SetScale("textlin");
        $graph->SetMarginColor('#e9e9e9');
        $graph->yaxis->scale->SetGrace(1);

        // Adjust the color for theshadow of the legend
        $graph->legend->SetShadow('darkgray@0.5');
        $graph->legend->SetFillColor('white');
        $graph->legend->SetFont(FF_VERDANA,FS_BOLD,8);

        $graph->yaxis->SetFont(FF_VERDANA,FS_NORMAL,8);
        $graph->yaxis->SetColor('black');

        $graph->ygrid->SetColor('#e9e9e9');

        // Setup graph title
        $graph->title->Set($title);
        // Some extra margin (from the top)
        $graph->title->SetMargin(3);
        $graph->title->SetFont(FF_VERDANA,FS_BOLD,11);
        if($ytitle)
        {
            $graph->yaxis->title->Set($ytitle);
            $graph->yaxis->title->SetMargin(3);
            $graph->yaxis->title->SetFont(FF_VERDANA,FS_BOLD,8);
        }

        $cbFmtPercentage_sum = 0;
        $count_bplot = count($bplot);
        for($i=0; $i<$count_bplot; $i++)
        {
            if($type_graph == 'line')
            {
                $bplots[$i] = new LinePlot($bplot[$i]['value']);
                $bplots[$i]->SetColor($color[$i%15]);
                $bplots[$i]->SetWeight(2);
                $graph->Add($bplots[$i]);
            }
            else
            {
                $bplots[$i] = new BarPlot($bplot[$i]['value']);

                if($count_xgrid < 2)
                {
                    $cbFmtPercentage_sum += $bplot[$i]['value'][0];
                    $bplots[$i]->value->SetFormatCallback("cbFmtPercentage");
                    $bplots[$i]->value->SetColor('black');
                    $bplots[$i]->value->Show();
                }

                $bplots[$i]->SetFillColor($color[$i%15]);
            }

            $bplot[$i]['legend'] = str_replace('\"', '"', $bplot[$i]['legend']);
            $bplots[$i]->SetLegend($bplot[$i]['legend']);
        }

        if($type_graph == 'line')
        {
            $graph->legend->Pos(0.02,0.02);
            $top = $count_bplot*17;
            $graph->img->SetMargin(55,20,$top,50);
            $graph->xaxis->SetLabelAngle(30);
        }
        else
        {
            $position = 45/$height;
            $graph->legend->Pos(0.02,$position);

            $gbarplot = new GroupBarPlot($bplots);
            $gbarplot->SetWidth(0.9);
            $graph->Add($gbarplot);

            if($count_xgrid > 1)
            {
                $graph->xaxis->SetLabelAngle(30);
                $graph->img->SetMargin(45,80,30,50);
            }
            else
                $graph->img->SetMargin(45,80,30,28);
        }

        $graph->xaxis->SetTickLabels($xgrid);
        $graph->xaxis->SetFont(FF_VERDANA,FS_BOLD,8);

        $graph->Stroke();
    }

    // вывод графика с указанием периода
    // $values - значения
    // $title - заголовок
    // $from - начало X-шкалы
    // $to - конец X-шкалы
    // $legends - легенда графика
    // возвращает массив 'title'- заголовок графика, 'height'- высота, 'width' - ширина, 'xgrid_str' - X-шкала, 'bplot' - значения
    function get_period_graph_data($values, $title, $from, $to, $legends = FALSE)
    {
        $from_t = mktime(0, 0, 0, substr($from, 5, 2), substr($from, 8, 2), substr($from, 0, 4));
        $to_t = mktime(0, 0, 0, substr($to, 5, 2), substr($to, 8, 2), substr($to, 0, 4));
        $day = 86400;
        $period = ($to_t - $from_t)/$day;
        if($period <= 30)
        {
            $step = 'day';
            for($i=0; $i<=$period; $i++)
            {
                $from_t_ = $from_t + $i*$day;
                $xgrids[$i] = date('Y-m-d', $from_t_);
                $xgrid_str[] = get_short_text_month(substr($xgrids[$i], 5, 2)) . ' ' . substr($xgrids[$i], 8, 2);
            }
        }
        elseif($period <= 400)
        {
            $values_ = $values;
            unset($values);
            foreach($values_ as $legend => $sub_values)
            {
                foreach($sub_values as $date => $count)
                {
                    $values[$legend][substr($date, 0, 7)] += $count;
                }
            }
            $step = 'month';
            $month = substr($from, 5, 2);
            $year = substr($from, 0, 4);
            $to_ = $to_t - 30*$day;
            for($i=0, $from_ = $from_t; $from_<=$to_t; $i++, $from_ = mktime(0, 0, 0, $month+$i, 1, $year))
            {
                $xgrids[$i] = date('Y-m', $from_);
                $xgrid_str[] = get_short_text_month(substr($xgrids[$i], 5, 2)) . ' ' . substr($xgrids[$i], 2, 2);
            }
        }
        else
        {
            $values_ = $values;
            unset($values);
            foreach($values_ as $legend => $sub_values)
            {
                foreach($sub_values as $date => $count)
                {
                    $values[$legend][substr($date, 0, 4)] += $count;
                }
            }
            $step = 'year';
            $year1 = substr($from, 0, 4);
            $year2 = substr($to, 0, 4);
            for($i=0; $i<=$year2-$year1; $i++)
            {
                $xgrids[$i] = $year1 + $i;
                $xgrid_str[] = $xgrids[$i];
            }
        }

        if(count($xgrids) < 2)
            return FALSE;

//        $values[$legend][$xgrid] = $count;

        if($legends === FALSE)
        {
            $legends = array_keys($values);
            asort($legends);
        }

        $count_xgrids = count($xgrids);
        $count_legend = count($legends);

        for($i=0, $d=0; $i<$count_legend; $i++)
        {
            for($j=0; $j<$count_xgrids; $j++)
            {
                if($values[$legends[$i]][$xgrids[$j]])
                    $bplot[$i]['value'][] = $values[$legends[$i]][$xgrids[$j]];
                else
                    $bplot[$i]['value'][] = 0;
                $d++;
            }
            $bplot[$i]['legend'] = $legends[$i];
        }

        if($count_legend > 15)
            $height = $count_legend*26;
        else
            $height = 300;
        if($count_xgrids > 10)
            $width = $count_xgrids*59;
        else
            $width=590;

        $return['title'] = $title;
        $return['height'] = $height;
        $return['width'] = $width;
        $return['xgrid_str'] = $xgrid_str;
        $return['bplot'] = $bplot;
        return $return;
    }

    // построение общего графика
    // $query - SQL запрос на получение данных для постороения графика
    // $title - заголовок
    // $xgrid - X-шкала
    // возвращает легенду и HTML код для вывода картинки
    function get_total_graph($query, $title, $xgrid)
    {
       global $SQL, $global_languages;
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        $sum = 0;
        for($i=0; $i<$num; $i++)
        {
            $legends[$i] = $SQL->result($result, $i, 'legend');
            if(!$legends[$i])
                $legends[$i] = $global_languages['UNDEFINED'];
            $count[$i] = $SQL->result($result, $i, 'count');
            $sum += $count[$i];
        }

        $bplot_value = '';
        $bplot_legend = '';
        for($i=0; $i<$num; $i++)
        {
            $legend[$i] = $legends[$i] . ' (' . round((100*$count[$i]/$sum), 1) .  '%)';
            if($i!=0)
            {
                $bplot_value .= ';';
                $bplot_legend .= ';';
            }
            $bplot_value .= $count[$i];
            $bplot_legend .= str_replace(';', ' ', $legend[$i]);
        }
        $bplot = '&bplot_value=' . rawurlencode($bplot_value) . '&bplot_legend=' . rawurlencode($bplot_legend);

        if($num>15)
            $height = '&height=' . $num*26;

        $result = array();
        $xgrid = '&xgrid=' . rawurlencode($xgrid);
        if($bplot)
            $result['img'] = $global_languages['TOTAL_PHP'].$sum.$global_languages['REGISTRATIONS_PHP'].'<br/><br/><img src="/admin/total_graph.php?title=' . rawurlencode($title) . $height . $xgrid . $bplot . '"><br><br>';
        else
            $result['img'] = '<b>' . $title . '.</b> Нет данных для построения графика!<br>';
        $result['legends'] = $legends;
        return $result;
    }

    // построение графика по периоду
    // $query - SQL запрос на получение данных для постороения графика
    // $title - заголовок
    // $from - начало X-шкалы
    // $to - конец X-шкалы
    // $legends - легенда графика
    // возвращает HTML код для вывода картинки
    function get_period_graph($query, $title, $from, $to, $legends = FALSE)
    {
        global $SQL, $global_languages;
        $from_t = mktime(0, 0, 0, substr($from, 5, 2), substr($from, 8, 2), substr($from, 0, 4));
        $to_t = mktime(0, 0, 0, substr($to, 5, 2), substr($to, 8, 2), substr($to, 0, 4));
        $day = 86400;
        $period = ($to_t - $from_t)/$day;
        if($period <= 30)
        {
            $step = 'day';
            $query .= ', xgrid';
            for($i=0; $i<$period; $i++)
            {
                $from_t_ = $from_t + $i*$day;
                $xgrids[$i] = date('Y-m-d', $from_t_);
            }
        }
        elseif($period <= 1000)
        {
            $step = 'month';
            $query .= ', substr(xgrid, 1, 7)';
            $month = substr($from, 5, 2);
            $year = substr($from, 0, 4);
            $to_ = $to_t - 30*$day;
            for($i=0, $from_ = $from_t; $from_<=$to_t; $i++, $from_ = mktime(0, 0, 0, $month+$i, 1, $year))
            {
                $xgrids[$i] = date('Y-m', $from_);
            }
        }
        else
        {
            $step = 'year';
            $query .= ', substr(xgrid, 1, 4)';
            $year1 = substr($from, 0, 4);
            $year2 = substr($to, 0, 4);
            for($i=0; $i<=$year2-$year1; $i++)
            {
                $xgrids[$i] = $year1 + $i;
            }
        }

        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++)
        {
            $legend = $SQL->result($result, $i, 'legend');
            if(!$legend)
                $legend = $global_languages['UNDEFINED'];

            $count = $SQL->result($result, $i, 'count');
            $xgrid = $SQL->result($result, $i, 'xgrid');
            if($step == 'month')
                $xgrid = substr($xgrid, 0, 7);
            elseif($step == 'year')
                $xgrid = substr($xgrid, 0, 4);
            $values[$legend][$xgrid] = $count;
        }

        if($legends === FALSE)
        {
            $legends = array_keys($values);
            asort($legends);
        }

        $bplot_value = '';
        $bplot_legend = '';
        $count_xgrids = count($xgrids);
        $count_legend = count($legends);

        for($i=0; $i<$count_legend; $i++)
        {
            if($i)
            {
                $bplot_value .= ';';
                $bplot_legend .= ';';
            }

            for($j=0; $j<$count_xgrids; $j++)
            {
                if($j!=0)
                    $bplot_value .= ':';
                if($values[$legends[$i]][$xgrids[$j]])
                    $bplot_value .= $values[$legends[$i]][$xgrids[$j]];
                else
                    $bplot_value .= 0;
            }
            $bplot_legend .= str_replace(';', ' ', $legends[$i]);
        }
        $bplot = '&bplot_value=' . rawurlencode($bplot_value) . '&bplot_legend=' . rawurlencode($bplot_legend);

        $xgrid_str = '';
        for($j=0; $j<$count_xgrids; $j++)
        {
            if($j)
                $xgrid_str .= ';';

            if($step == 'month')
                $xgrid_str .= get_short_text_month(substr($xgrids[$j], 5, 2)) . ' ' . substr($xgrids[$j], 2, 2);
            elseif($step == 'day')
                $xgrid_str .= get_short_text_month(substr($xgrids[$j], 5, 2)) . ' ' . substr($xgrids[$j], 8, 2);
        }
        $xgrid_str = '&xgrid=' . rawurlencode($xgrid_str);

        if($count_legend > 15)
            $height = '&height=' . $count_legend*26;
        if($count_xgrids > 10)
            $width = '&width=' . $count_xgrids*59;

        if($bplot)
            return '<img src="/lkadm/total_graph.php?title=' . rawurlencode($title) . $height . $width . $xgrid_str . $bplot . '"><br><br>';
        else
            return '<b>' . $title . '.</b>'.$global_languages['NO_DATA_FOR_GRAPH'].'<br>';
    }

    // $date - месяц
    // возвращает месяц в виде текста три символа
    function get_short_text_month($date)
    {
       global $global_languages;
        if(strlen($date) == 1)
            $date = '0' . $date;
        $monthlist = array(
                     '01' => $global_languages['SHORT_JANUARY'],
                     '02' => $global_languages['SHORT_FEBRUARY'],
                     '03' => $global_languages['SHORT_MARCH'],
                     '04' => $global_languages['SHORT_APRIL'],
                     '05' => $global_languages['SHORT_MAY'],
                     '06' => $global_languages['SHORT_JUNE'],
                     '07' => $global_languages['SHORT_JULY'],
                     '08' => $global_languages['SHORT_AUGUST'],
                     '09' => $global_languages['SHORT_SEPTEMBER'],
                     '10' => $global_languages['SHORT_OCTOBER'],
                     '11' => $global_languages['SHORT_NOVEMBER'],
                     '12' => $global_languages['SHORT_DECEMBER']
                     );

        return $monthlist[$date];
    }

    // $i - номер строчки таблицы
    // возвращает класс строчки таблицы
    function getclass($i)
    {
        if($i%2)
            $class = 'tr_1';
        else
            $class = 'tr_2';
        return $class;
    }

    // $send - статус
    // возвращает HTML код статуса
    function get_state($send)
    {
       global $global_languages;
        if($send == 'deliver')
        {
            $return['text'] = $global_languages['DELIVER'];
            $return['html'] = '<span class="g">'.$global_languages['DELIVER'].'</span>';
        }
        elseif($send == 'not_deliver')
        {
            $return['text'] = $global_languages['NOT_DELIVER'] ;
            $return['html'] = '<span class="r">'.$global_languages['NOT'] .'&nbsp;'.$global_languages['DELIVER'].'</span>';
        }
        elseif($send == 'expired')
        {
            $return['text'] = $global_languages['EXPIRED'];
            $return['html'] = '<span class="r">'.$global_languages['EXPIRED'].'</span>';
        }
        else
        {
            $return['text'] = $global_languages['SENDED'];
            $return['html'] = '<span class="b">'.$global_languages['SENDED'].'</span>';
        }
        return $return;
    }

    // $id_user - ID пользователя
    // $count_sms - количество частей
    // $phone - номер телефона получателя SMS
    // $originator - отправитель
    // возвращает конечного агрегатора для конкретной SMS
    function get_id_aggregating($id_user, $id_partner, $MCC,$MNC, $type_sms,$originator, $count_sms,$turn_id_sms=FALSE)
    {
        global $SQL;
        $id_aggregating_result = 0;
        $originator = stripslashes($originator);
        if($id_partner)
           $where_partner=' or (id_client=-' . $id_partner . ' and MCC="'.$MCC.'" and (MNC="'.$MNC.'" or MNC=0)) and (BINARY "'.addslashes($originator).'" like originator OR (
(

SELECT ssto.id_group_originator
FROM sms_scheme_traffic_originator AS ssto
WHERE BINARY  "'.addslashes($originator).'" = originator
AND sms_scheme_traffic.id_group_originator = ssto.id_group_originator
)
AND originator =  "" AND `id_group_originator`>0
)
OR (originator =  "" AND `id_group_originator`=0)
)';
        else
           $where_partner='';
        if($turn_id_sms!==FALSE)
        {
              $query_used_aggregating = 'select id_aggregating from sms_used_aggregating where id_turn="'.$turn_id_sms.'"';
              $result_used_aggregating = $SQL->query($query_used_aggregating);
              while($res=$SQL->fetch_assoc($result_used_aggregating))
              {
                  $used[$res['id_aggregating']]=1;
              }
        }
        $query = 'select sql_cache id_group_aggregating  from sms_scheme_traffic where ((id_client="' . $id_user . '" or id_client=0) and MCC="'.$MCC.'" and (MNC="'.$MNC.'" or MNC=0)) and (BINARY "'.addslashes($originator).'" like originator OR (
(

SELECT ssto.id_group_originator
FROM sms_scheme_traffic_originator AS ssto
WHERE BINARY  "'.addslashes($originator).'" = originator
AND sms_scheme_traffic.id_group_originator = ssto.id_group_originator
)
AND originator =  "" AND `id_group_originator`>0
)
OR (originator =  "" AND `id_group_originator`=0)
) '.$where_partner.' and type_sms="'.$type_sms.'" order by if(id_client>0, 1, if(id_client<0, 2, 3)) asc, priority asc, MNC desc';
        $result_aggregating = $SQL->query($query);
        $num_aggregating = $SQL->num_rows($result_aggregating);
        for($i=0; $i<$num_aggregating; $i++)
        {
                $id_group_aggregating = $SQL->result($result_aggregating, $i, 'id_group_aggregating');
                $query = 'select sql_cache sd.id_aggregating, sd.percent, sd.main from sms_distribution as sd, sms_aggregating as sa where sd.id_aggregating=sa.id_aggregating and sd.id_group_aggregating="' . $id_group_aggregating . '" and switch="on" and sa.works="1" and sa.cope="1" and sa.num_sms>=' . $count_sms;
                $result = $SQL->query($query);
                $num = $SQL->num_rows($result);
                $percent = 0;
                $percent_main=0;
                unset($aggregating);
                unset($aggregating_main);
                for($j=0; $j<$num; $j++)
                {
                    $id_aggregating = $SQL->result($result, $j, 'sd.id_aggregating');
                    if($used[$id_aggregating])
                       continue;
                    $main=$SQL->result($result, $j, 'sd.main');
                    if($main)
                    {
                       $percent_main+= $SQL->result($result, $j, 'sd.percent');
                       $aggregating_main[$j]['id_aggregating'] = $id_aggregating;
                       $aggregating_main[$j]['percent'] = $percent_main;
                    }
                    else
                    {
                       $percent += $SQL->result($result, $j, 'sd.percent');
                       $aggregating[$j]['id_aggregating'] = $id_aggregating;
                       $aggregating[$j]['percent'] = $percent;
                    }
                }
                if(is_array($aggregating_main))
                {
                    $rand = rand(1, $percent_main);
                    foreach($aggregating_main as $am_value)
                    {
                        if($rand <= $am_value['percent'])
                        {
                            $id_aggregating_result = $am_value['id_aggregating'];
                            break 2;
                        }
                    }
                }
                else
                {
                    if(is_array($aggregating))
                      {
                           $rand = rand(1, $percent);
                             foreach($aggregating as $agg_value)
                             {
                                 if($rand <= $agg_value['percent'])
                                 {
                                     $id_aggregating_result = $agg_value['id_aggregating'];
                                     break 2;
                                 }
                             }
                      }
                }
        }

        if($id_aggregating_result)
        {
            $query = 'update sms_aggregating set num_sms=num_sms-' . $count_sms . ' where  id_aggregating!=40 AND  id_aggregating="' . $id_aggregating_result . '"';
            $SQL->query_async($query);
        }
        return $id_aggregating_result;
    }

    /**
     * Запись SMS в очередь на отправку или в запланированную очередь
     * @global type $SQL
     * @param type $destination_addr адрес получателя
     * @param type $send_from_date дата отправки
     * @param type $send_from с какого времени отправлять
     * @param type $id_user ID пользователя
     * @param type $id_control_text
     * @param type $id_package_user ID пакета
     * @param type $id_partner
     * @param type $id_package_partner
     * @param type $MCC
     * @param type $MNC
     * @param type $MNC_user
     * @param type $MNC_partner
     * @param type $source_addr отправитель
     * @param type $type
     * @param type $short_message текст сообщения
     * @param type $num_messages число частей SMS
     * @param type $id_currency
     * @param type $sell_we
     * @param type $sell_partner
     * @param type $buy_partner
     * @param type $id_aggregating ID агрегатора
     * @param type $sms_priority приоритет SMS
     * @param type $validity_period время жизни SMS
     * @param type $id_name_delivery
     * @return int ID SMS в очереди
     */
    function set_sms_turn($destination_addr, $send_from_date, $send_from, $id_user, $id_control_text, $id_package_user,  $id_partner, $id_package_partner, $MCC, $MNC, $MNC_user, $MNC_partner, $source_addr, $type, $short_message, $num_messages, $id_currency, $sell_we, $sell_partner ,$buy_partner , $id_aggregating, $sms_priority, $validity_period = '', $id_name_delivery = 0)
    {
        global $SQL;
        if($validity_period)
            $validity_period = ', validity_period="' . $validity_period . '"';
        else
            $validity_period = '';

        $date = date('Y-m-d');
        $hour = date('H:i');
        $hour = explode(':', $hour);
        $from = explode(':', $send_from);

        $id_aggregating_hlr = get_id_aggregating_hlr($id_user, $id_partner, $MCC,$MNC, $type, $source_addr, $num_messages);
        $status_HLR = 0;
        if($id_aggregating_hlr)
            $status_HLR=1;

        if($id_control_text || (strcmp($send_from_date, $date) == 0 && ($from[0] > $hour[0] || ($from[0] == $hour[0] && $from[1] > $hour[1]))) || strcmp($send_from_date, $date) > 0)
        {
            $query = 'insert into sms_turn_time set id_user="' . $id_user . '", id_package_user="' . $id_package_user . '",  id_partner="' . $id_partner . '", id_package_partner="' . $id_package_partner . '", MCC="'.$MCC.'", MNC ="'.$MNC.'", MNC_user ="'.$MNC_user.'", MNC_partner="'.$MNC_partner.'" , originator="' . $source_addr . '", type_sms="' . $type . '", text_sms="' . $short_message . '", count_sms="' . $num_messages . '", id_currency="' . $id_currency . '", sell_we="' . $sell_we . '", sell_partner="' . $sell_partner . '", buy_partner="' . $buy_partner . '", phone="' . $destination_addr . '", id_script="' . rand(1, NUM_SEND_SCRIPT) . '", id_aggregating="' . $id_aggregating . '", send_from="' . $send_from . ':00", sms_priority="' . $sms_priority . '", send_from_date="' . $send_from_date . '", id_name_delivery="' . $id_name_delivery . '", id_control_text="' . $id_control_text . '"' . $validity_period;
            $SQL->query($query);
            $id_phone = -1 * $SQL->insert_id();
        }
        else
        {
            $query = 'insert into sms_turn set id_user="' . $id_user . '", id_package_user="' . $id_package_user . '",  id_partner="' . $id_partner . '", id_package_partner="' . $id_package_partner . '", MCC="'.$MCC.'", MNC ="'.$MNC.'", MNC_user ="'.$MNC_user.'", MNC_partner="'.$MNC_partner.'", originator="' . $source_addr . '", type_sms="' . $type . '", text_sms="' . $short_message . '", count_sms="' . $num_messages . '", id_currency="' . $id_currency . '", sell_we="' . $sell_we . '", sell_partner="' . $sell_partner . '", buy_partner="' . $buy_partner . '", phone="' . $destination_addr . '", id_script="' . rand(1, NUM_SEND_SCRIPT) . '", id_aggregating="' . $id_aggregating . '", sms_priority="' . $sms_priority . '", id_name_delivery="' . $id_name_delivery . '", status_HLR="'.$status_HLR.'"' . $validity_period;
            $SQL->query($query);
            $id_phone = $SQL->insert_id();
            if($id_aggregating_hlr){
                $query = 'insert into sms_turn_HLR set id_turn="'.$id_phone.'", originator="' . $source_addr . '", phone="' . $destination_addr . '", type_sms="' . $type . '", text_sms="' . $short_message . '", sms_priority="' . $sms_priority . '", id_aggregating_HLR="' . $id_aggregating_hlr . '"';
                $SQL->query($query);
            }
        }
        
        return $id_phone;
    }

    function get_id_name_delivery($name_delivery, $id_user)
    {
        global $SQL;
        if($name_delivery)
        {
            global $GLOBAL_CACHE_DATA;
            if(isset($GLOBAL_CACHE_DATA['get_id_name_delivery'][$id_user][$name_delivery]))
                return $GLOBAL_CACHE_DATA['get_id_name_delivery'][$id_user][$name_delivery];
            else
            {
                $query = 'select id_name_delivery from sms_name_delivery where id_user="' . $id_user . '" and name_delivery="' . addslashes($name_delivery) . '"';
                $result = $SQL->query($query);
                if($SQL->num_rows($result))
                    $id_name_delivery = $SQL->result($result, 0, 'id_name_delivery');
                else
                {
                    $query = 'insert into sms_name_delivery set name_delivery="' . addslashes($name_delivery) . '", id_user="' . $id_user . '"';
                    $SQL->query($query);
                    $id_name_delivery = $SQL->insert_id();
                }
                $GLOBAL_CACHE_DATA['get_id_name_delivery'][$id_user][$name_delivery] = $id_name_delivery;
                return $id_name_delivery;
            }
        }
        else
            return 0;
    }

    function get_gradual($phone, $local, $gradual, $local_time)
    {
        global $from, $from_minute, $from_send, $from_minute_send, $to, $to_minute, $date_send_sms, $date_send_from, $date_send_to;
        if($gradual == 'on')
        {
            $from_minute_send++;
            if($from_minute_send >59)
            {
                $from_send++;
                $from_minute_send = 0;
            }
            if($from_send > $to || ($from_send == $to && $from_minute_send >= $to_minute))
            {
                $from_send = $from;
                $from_minute_send = $from_minute;
                $date_send_sms = date_add_day($date_send_sms, 1);
                if(strcmp($date_send_sms, $date_send_to) > 0)
                    $date_send_sms = $date_send_from;
            }
            $time[0] = $date_send_sms;
            $time[1] = add_zero($from_send, 2) . ':' . add_zero($from_minute_send, 2);
        }
        else
        {
            $time[0] = $date_send_from;
            $time[1] = add_zero($from, 2) . ':' . add_zero($from_minute, 2);
        }

        if($local)
            if(($local_time_ = get_time_zone($phone))!==FALSE)
                $local_time=$local_time_;

        $tmp = date_add_minute($time[0] . ' ' . $time[1] . ':00', -$local_time);
        $time[0] = substr($tmp, 0, 10);
        $time[1] = substr($tmp, 11, 5);
        return $time;
    }

    function get_allow_time($time, $user_send_from, $user_send_to)
    {
        $send_from_date = substr($time, 0, 10);
        $send_from = substr($time, 11, 5);
        $from = explode(':', $send_from);
        if($from['0'] < $user_send_from)
            $from = array(0 => $user_send_from, 1 => 0);
        elseif($from['0'] >= $user_send_to)
        {
            $from = array(0 => $user_send_from, 1 => 0);
            $send_from_date = date_add_day($send_from_date, 1);
        }
        return $send_from_date . ' ' . $from[0] . ':' . $from[1];
    }

    // $b - если было введено значения которое нужно изменить
    function add_aggregating_pay($id_aggregating,$MCC,$MNC,$price,$b)
    {
        global $SQL;
        if($b)
        {
            $query = 'UPDATE sms_aggregating_price SET price="'.$price.'" WHERE id_aggregating="'.$id_aggregating.'" and MCC="'.$MCC.'" and MNC="'.$MNC.'"';
            $result = $SQL->query($query);
            if($SQL->affected_rows() == 0)
            {
                $query = 'insert into sms_aggregating_price set price="' . $price . '", id_aggregating="' . $id_aggregating . '", MCC="' . $MCC . '", MNC="' .$MNC . '"';
                $SQL->query($query);
            }
        }else
        {
            $query = 'SELECT price_sms FROM sms_aggregating WHERE id_aggregating="'.$id_aggregating.'"';
            $result = $SQL->query($query);
            $res = $SQL->fetch_assoc($result);
            if($res && $res['price_sms'])
            {
                $price = $res['price_sms'];
                $query = 'SELECT * FROM sms_aggregating_price WHERE id_aggregating="'.$id_aggregating.'" and MCC="'.$MCC.'" and MNC="'.$MNC.'"';
                $result = $SQL->query($query);
                if(!($res = $SQL->fetch_assoc($result)))
                {
                    $query = 'insert into sms_aggregating_price set price="' . $price . '", id_aggregating="' . $id_aggregating . '", MCC="' . $MCC . '", MNC="' .$MNC . '"';
                    $SQL->query($query);
                }
            }
        }
    }

    function set_aggregating_pay($id_aggregating = FALSE)
    {
        global $SQL, $global_aggregating_pay;
        $query = 'SELECT sql_cache * FROM sms_aggregating_price where id_aggregating="' . $id_aggregating . '"';
        if(!$result = $SQL->query($query)) return;
        $global_aggregating_pay = array();
        while($res = $SQL->fetch_assoc($result))
        {
            $global_aggregating_pay[$res['MCC']][$res['MNC']] = array('id_currency'=>$res['id_currency'], 'price_sms'=>$res['price_sms'], 'deliver_payment'=>$res['deliver_payment']);
        }
    }
    function get_aggregating_pay($MCC,$MNC)
    {
        global $global_aggregating_pay;
        if(isset($global_aggregating_pay[$MCC][$MNC]))
            return $global_aggregating_pay[$MCC][$MNC];
        if(isset($global_aggregating_pay[$MCC]['0']))
            return $global_aggregating_pay[$MCC]['0'];
        return FALSE;
    }

     function set_param_send_sms()
     {
         set_time_zone();
         set_control();
         set_phone_code_in_array();
         set_originator_in_array();
     }

    function set_phone_code_in_array()
    {
        $def_redis = set_phone_code_in_array_redis();
        if(!$def_redis){
            global $SQL, $phone_codes_array, $phone_codes_array_sql;
            $phone_codes_array = array();
            $query = 'select spc.phone_code,spct.id_phone_code,spct.MCC,spct.MNC,spct.length from sms_phone_code_title as spct, sms_phone_code as spc where spct.id_phone_code=spc.id_phone_code';
            $result = $SQL->unbuffered_query($query);
            while($res=$SQL->fetch_assoc($result))
            {
                $len_send = strlen($res['phone_code']);
                $tmp = &$phone_codes_array;
                for($j=0; $j<$len_send; $j++)
                {
                    $tmp = &$tmp[$res['phone_code'][$j]];
                }
                $tmp['v'] = array('MCC' => $res['MCC'], 'MNC' => $res['MNC'], 'length' => $res['length']);
            }
            $phone_codes_array_sql = time();
        }
    }
    
    function set_phone_code_in_array_redis(){
        global $phone_codes_array, $redis3, $REDIS3_DIE;
        if(!$REDIS3_DIE){
            if($redis3->dbSize() > 1000){
                $phone_codes_array2 = $redis3->get('phone_codes_array');
                if($phone_codes_array2!=''){
                    $phone_codes_array = unserialize($phone_codes_array2);
                    if(count($phone_codes_array)>0){
                        return true;
                    } else {
                        return false;
                    }
                } else {
                    return false;
                }
            } else {
                return false;
            }
        } else {
            return false;
        }
    }

    function get_phone_code_from_array($phone,$not_def=array())
    {
        global $phone_codes_array;
        $phone = (string)$phone;
        $len_phone = strlen($phone);
        $tmp = &$phone_codes_array;
        $MCC = 0;
        $MNC = 0;
        for($j=0; $j<$len_phone; $j++)
        {
            if(is_array($tmp))
            {
                $tmp = &$tmp[$phone[$j]];
                if(isset($tmp['v']) and !$not_def[$tmp['v']['MCC'].'-'.$tmp['v']['MNC']])
                {
                    $MCC = $tmp['v']['MCC'];
                    $length = $tmp['v']['length'];
                    if($tmp['v']['MNC'])
                        $MNC = $tmp['v']['MNC'];
                }
            }
            else
                break;
        }
        if($length){
            if($len_phone!=$length){
                $MCC = 0;
                $MNC = 0;
            }
        }
        /*$error_def = 0;
        if($error_def){
            $not_def[$MCC.'-'.$MNC] = 1;
            $mcc_mnc_len = get_phone_code_from_array($phone, $not_def);
            $MCC = $mcc_mnc_len["MCC"];
            $MNC = $mcc_mnc_len["MNC"];
        }*/
        $mcc_mnc = get_port_phone($phone);
        if($mcc_mnc)
            return $mcc_mnc;
        return array('MCC' => $MCC, 'MNC' => $MNC); 
    }
    
    function get_port_phone($phone=false)
    {
        global $redis3, $change_prt_phone_code, $phone_codes_array_sql, $REDIS3_DIE;
        $return = false;
        if($phone){
            $time2 = time() - 5;
            if($phone_codes_array_sql > $time2) {
                return false;
            } elseif(!$REDIS3_DIE){
                $phone_port_redis = $redis3->get('p-'.$phone);
                if($phone_port_redis){
                    $phone_port_redis_explode = explode(":",$phone_port_redis);
                    $return = array('MCC' => $phone_port_redis_explode['0'], 'MNC' => $phone_port_redis_explode['1']);
                }
            } else {
                $redis3 = new Redis();
                $redis3->connect('localhost', 6379) or $REDIS3_DIE='1';
                $redis3->select(3);
                $phone_codes_array_sql = 0;
                set_phone_code_in_array();
            }
        }
        return $return;
    }
    
    function set_change_prt_phone_code()
    {
        global $SQL, $change_prt_phone_code;
        $change_prt_phone_code = array();
        $query = 'select * from sms_prt_operators order by MCC, MNC';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num){
            for($j=0; $j<$num; $j++)
            {
                $sms_prt_operators = $SQL->fetch_assoc($result, $j);
                $change_prt_phone_code[$sms_prt_operators['mcc_prt']][$sms_prt_operators['mnc_prt']]=array('MCC'=>$sms_prt_operators['MCC'],'MNC'=>$sms_prt_operators['MNC']);
            }
        }
        return $change_prt_phone_code;
    }

    function set_time_zone()
    {
        global $SQL, $time_zones_array, $redis3, $REDIS3_DIE;
        if(!$redis3->exists('time_zones_array') OR $REDIS3_DIE){
            $time_zones_array = array();
            $query = 'select time_zone, code_from, code_to from sms_def_code';
            $result = $SQL->query($query);
            $num_sms = $SQL->num_rows($result);
            for($i=0; $i<$num_sms; $i++)
            {
                $send_from = $SQL->result($result, $i, 'code_from');
                $time_zone = $SQL->result($result, $i, 'time_zone');
                $send_to = $SQL->result($result, $i, 'code_to');
                $len_send = strlen($send_from);
                while(substr($send_from, $len_send-1, 1) == '0' && substr($send_to, $len_send-1, 1) == '9')
                    $len_send--;

                $send_from = substr($send_from, 0, $len_send);
                $send_to = substr($send_to, 0, $len_send);

                for(; $send_from <= $send_to; $send_from++)
                {
                    $tmp = &$time_zones_array;
                    for($j=0; $j<$len_send; $j++)
                    {
                        $sf = substr($send_from, $j, 1);
                        $st = substr($send_to, $j, 1);
                        $next_sf = substr($send_from, $j+1, 1);
                        $tmp = &$tmp[$sf];
                        if($next_sf == 0 && $sf != $st)
                        {
                            $send_from += pow(10, $len_send - $j - 1) - $send_from%(pow(10, $len_send - $j - 1)) - 1;
                            break;
                        }
                    }
                    $tmp = $time_zone;
                }
            }
            if(!$REDIS3_DIE){
                $redis3->set('time_zones_array', json_encode($time_zones_array, JSON_UNESCAPED_UNICODE));
            }
        } else {
            $time_zones_array = json_decode($redis3->get('time_zones_array'), true);
        }
    }

    function get_time_zone($phone)
    {
        global $time_zones_array;
        $phone = (string)$phone;
        $len_phone = strlen($phone);
        $tmp = &$time_zones_array;
        for($j=0; $j<$len_phone; $j++)
        {
            $sp = substr($phone, $j, 1);
            if(isset($tmp) && is_array($tmp) && isset($tmp[$sp]))
                $tmp = &$tmp[$sp];
            else
                break;
        }
        if(!isset($tmp) || is_array($tmp) || !$tmp)
            return FALSE;
        else
            return $tmp*60;
    }

    function set_smpp_error($id_aggregating)
    {
        global $SQL, $smpp_errors_array;
        $smpp_errors_array = array();
        $query = 'select id_smpp_error, code from sms_smpp_error_aggregating where id_aggregating="' . $id_aggregating . '"';
        $result = $SQL->query($query);
        $num_sms = $SQL->num_rows($result);
        for($i=0; $i<$num_sms; $i++)
        {
            $id_smpp_error = $SQL->result($result, $i, 'id_smpp_error');
            $code = $SQL->result($result, $i, 'code');
            $smpp_errors_array[$code] = $id_smpp_error;
        }
    }

    function get_smpp_error($code)
    {
        global $smpp_errors_array;
        if(isset($smpp_errors_array[$code]))
            return $smpp_errors_array[$code];
        else
            return FALSE;
    }

    // преобразование даты+время в число секунд
    // $date - дата+время
    // возвращает число секунд для указанной даты
    function get_time($date)
    {
        return mktime(substr($date, 11, 2), substr($date, 14, 2), substr($date, 17, 2), substr($date, 5, 2), substr($date, 8, 2), substr($date, 0, 4));
    }

    // Заполнение строки нулями. нули добавляются в начало строки
    // $number - строка которую нужно дополнить нулями
    // $num - длина на которую нужно заполнить строку нулями
    // Возвращает строку заполненную нулями. Если длина строки больше $num, то строка обрежется до $num символов
    function add_zero($number, $num)
    {
        $len = strlen($number);
        if($len > $num)
        {
            $number = substr($number, -$num);
            $len = $num;
        }
        for($i=0; $i<$num-$len; $i++)
        {
            $number = '0' . $number;
        }
        return $number;
    }

    // $text_sms - текст sms
    // возвращает количество частей SMS
    function get_count_sms($type, $text_sms)
    {
        $text_sms = stripslashes($text_sms);
        if($type == 'sms' || $type == 'flashsms')
        {
            $encoding = get_char_message($text_sms);
            if($encoding == 0x8)
            {
                $len_text_sms = mb_strlen($text_sms);
                if($len_text_sms > 70)
                    $count_sms = ceil(($len_text_sms/67));
                else
                    $count_sms = 1;
            }
            else
            {
                $len_text_sms1 = strlen(convert_encoding($text_sms, 0x0, TRUE)); // GSM
                $len_text_sms2 = strlen(convert_encoding($text_sms, 0x3, TRUE)); // Латиница
                if($len_text_sms1 > $len_text_sms2)
                    $len_text_sms = $len_text_sms1;
                else
                    $len_text_sms = $len_text_sms2;

                if($len_text_sms > 160)
                    $count_sms = ceil(($len_text_sms/153));
                else
                    $count_sms = 1;
            }
        }
        elseif($type == 'wappush')
        {
            $tmp = get_param_sms($text_sms);
            $url = $tmp[0];
            $text = $tmp[1];
            $len_text = strlen(set_url_wap_push($url)) + strlen($text) + 15;
            if($len_text <= 133)
                $count_sms = 1;
            else
            {
                $max_len = 128;
                $count_sms = ceil($len_text / $max_len);
            }
        }
        elseif($type == 'vcard')
        {
            $text_sms = get_vcard($text_sms);
            $len_text = strlen($text_sms);
            if($len_text <= 133)
                $count_sms = 1;
            else
            {
                $max_len = 128;
                $count_sms = ceil($len_text / $max_len);
            }
        }
        return $count_sms;
    }

    // преобразование кодировки
    // $from - из какой кодировки
    // $to - в какую кодировку преобразовать строку
    // $str - строка
    // возвращает строку в $to кодировке
    function change_charset($from, $to, $str)
    {
        $str = mb_convert_encoding($str, $to, $from);
        return $str;
    }

    // проверка наличия пакета
    // $id_user - ID пользователя
    // $date - дата
    // возвращает TRUE если существует пакет на заданную дату, FALSE в противном случае
    function check_package_date($id_user, $date)
    {
        global $SQL;
        $query = 'select id_package_user from sms_package_user where id_user="' . $id_user . '" and date_start="' . $date . '"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
            return TRUE;
        else
            return FALSE;
    }

    function get_rest_packages($id, $date, $table = 'user')
    {
        global $SQL;
        if($table == 'user')
            $ful_table = 'sms_package_user';
        else
            $ful_table = 'sms_package_partner';
        $return = array();
        $query = 'select pu.id_package_' . $table . ', pu.limit_price, pu.limit_price-pu.waste as rest, pu.credit from ' . $ful_table . ' as pu where pu.id_' . $table . '="' . $id . '" and pu.date_start<="' . $date . '" and pu.date_stop>="' . $date . '" order by pu.date_start asc';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        $return['money'] = 0;
        $return['credits'] = 0;
        for($i=0; $i<$num; $i++)
        {
            $limit_price = $SQL->result($result, $i, 'pu.limit_price');
            if($limit_price == 0)
                $return['unlimited'] = TRUE;
            $id_package = $SQL->result($result, $i, 'pu.id_package_' . $table);
            $rest = $SQL->result($result, $i, 'rest');
            $credit = $SQL->result($result, $i, 'pu.credit');
            $return['money'] += $rest;
            $return['credits'] += $credit;

            $query = 'select sp.price_sms, spct.title from sms_price_' . $table . ' as sp left join sms_phone_code_title as spct on(sp.MCC=spct.MCC and sp.MNC=spct.MNC) where sp.id_package_' . $table . '="' . $id_package . '"';
            $result_price = $SQL->query($query);
            $num_price = $SQL->num_rows($result_price);
            for($j=0; $j<$num_price; $j++)
            {
                $price_sms = $SQL->result($result_price, $j, 'sp.price_sms');
                if($price_sms)
                {
                    $title = $SQL->result($result_price, $j, 'spct.title');
                    if(isset($return['sms'][$title]))
                        $return['sms'][$title] += round($rest/$price_sms);
                    else
                        $return['sms'][$title] = round($rest/$price_sms);
                }
            }
            $return['money'] = round($return['money'], 2);
            $return['credits'] = round($return['credits'], 2);
        }
        return $return;
    }

    function get_rest_packages_all($id, $date, $table = 'user')
    {
        global $SQL, $global_languages;
        if($table == 'user')
            $ful_table = 'sms_package_user';
        else
            $ful_table = 'sms_package_partner';
        $return = array();
        $query = 'select pu.id_package_' . $table . ', pu.limit_price, pu.limit_price-pu.waste as rest from ' . $ful_table . ' as pu where pu.id_' . $table . '="' . $id . '" and pu.date_start<="' . $date . '" and pu.date_stop>="' . $date . '" order by pu.date_start asc';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        $return['money'] = 0;
        for($i=0; $i<$num; $i++)
        {
            $limit_price = $SQL->result($result, $i, 'pu.limit_price');
            if($limit_price == 0)
                $return['unlimited'] = TRUE;
            $id_package = $SQL->result($result, $i, 'pu.id_package_' . $table);
            $rest = $SQL->result($result, $i, 'rest');
            $return['money'] += $rest;

            $query = 'select price_sms, MCC, MNC  from sms_price_' . $table . '  where id_package_' . $table . '="' . $id_package . '"';
            $result_price = $SQL->query($query);
            $num_price = $SQL->num_rows($result_price);
            for($j=0; $j<$num_price; $j++)
            {
                $price_sms = $SQL->result($result_price, $j, 'price_sms');
                if($price_sms)
                {
                    $MCC = $SQL->result($result_price, $j, 'MCC');
                    $MNC = $SQL->result($result_price, $j, 'MNC');
                    if(isset($return['sms'][$MCC][$MNC]))
                        $return['sms'][$MCC][$MNC] += round($rest/$price_sms);
                    else
                        $return['sms'][$MCC][$MNC] = round($rest/$price_sms);
                }
            }
            $return['money'] = round($return['money'], 2);
        }

        $query='select if(MNC=0,-1000,sp.priority)as priority,if(priority_ is NULL,sp.priority,priority_) as priority_, title,MCC, MNC from sms_phone_code_title as sp LEFT JOIN (Select MCC as MCC_,priority as priority_ FROM sms_phone_code_title where MNC=0) as spct ON(spct.MCC_=sp.MCC) order by priority_ asc,sp.MCC,priority asc, title asc';
        $result = $SQL->query($query);
        $sms = array();
        $title_MCC = array();
        $sms_others=array();
        $sms_others[0]=array('title'=>$global_languages['OTHERS']);
        $b_others=FALSE;
        while($res=$SQL->fetch_assoc($result))
        {
            $res['title']=get_translate_phone_code_title($res['title']);
            if($res['MNC']==0)
                $title_MCC[$res['MCC']]=$res['title'];
            if(isset($title_MCC[$res['MCC']]))
            {
                if(isset($return['sms'][$res['MCC']][$res['MNC']]))
                {
                    if(!isset($sms[$res['MCC']][0]))
                        $sms[$res['MCC']][0]=array('title'=>$title_MCC[$res['MCC']]);
                    $sms[$res['MCC']][$res['MNC']]=array('title'=>$res['title'],'count'=>$return['sms'][$res['MCC']][$res['MNC']]);
                }
            }else
                if(isset($return['sms'][$res['MCC']][$res['MNC']]))
                {
                    if(!$b_others)$b_others=TRUE;
                    $sms_others[]=array('title'=>$res['title'],'count'=>$return['sms'][$res['MCC']][$res['MNC']]);
                }
        }
        if($b_others)
            $sms[]=$sms_others;

        $return['sms']=$sms;
        return $return;
    }

    function return_sms($id, $table, $id_user = 0)
    {
        global $SQL;
        if($id_user)
            $where = ' and id_user="' . $id_user . '"';
        $query = 'select count_sms, id_package_user, sell_we, sell_partner, id_package_partner from '.$table.'  where id_turn="' . $id . '"' . $where;
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $sell_we = $SQL->result($result, 0, 'sell_we');
            $sell_partner = $SQL->result($result, 0, 'sell_partner');
            $id_package_user = $SQL->result($result, 0, 'id_package_user');
            $id_package_partner = $SQL->result($result, 0, 'id_package_partner');
            $count_sms = $SQL->result($result, 0, 'count_sms');

            if($id_package_partner)
            {
                $query = 'update sms_package_partner set waste=waste-'.$sell_we.' where id_package_partner="'.$id_package_partner.'"';
                $SQL->query($query);
                $sell_we=$sell_partner;
            }
            $query = 'update sms_package_user set waste=waste-'.$sell_we.' where id_package_user="'.$id_package_user.'"';
            $SQL->query($query);
            $query = 'delete from ' . $table . ' where id_turn="' . addslashes($id) . '"';
            $SQL->query($query);
            $query = 'delete from sms_state_final where id_sms="' . addslashes($id) . '"';
            $SQL->query($query);
            return $count_sms;
        }
        else
            return 0;
    }
    function sms_final_pay_control($id_turn)
    {
        global $SQL;
        set_convertion_currency();
        $date = date('Y-m-d H:i:s');
        $query = 'insert into sms_send_trafic (id_sms, id_user, id_package_user, id_partner, id_package_partner, MCC, MNC, MNC_user, MNC_partner, time_put_turn, originator, phone, type_sms, text_sms, num_parts, id_name_delivery, validity_period, time) (select -1*cast(id_turn as signed), id_user, id_package_user, id_partner, id_package_partner,  MCC,MNC , MNC_user, MNC_partner, time_put_turn, originator, phone, type_sms, text_sms, count_sms, id_name_delivery, validity_period,"'.$date.'" from sms_turn_time where id_turn="'.$id_turn.'")';
        $SQL->query($query);
        if($SQL->affected_rows() > 0)
        {
            $query = 'SELECT * from sms_turn_time where id_turn="'.$id_turn.'"';
            $result = $SQL->query($query);
            if($res = $SQL->fetch_assoc($result))
            {
                $query = 'delete from sms_turn_time where id_turn="'.$id_turn.'" and id_control_text!=0';
                $SQL->query($query);
                $query = 'UPDATE sms_state_final SET state="not_deliver", final="1" where id_sms="-'.$id_turn.'"';
                $SQL->query($query);
                $date = substr($date, 0, 13) . ':00:00';
                $param = get_param_id_user($res['id_user']);
                update_sms_stat($res['id_user'],$res['id_partner'],$param['id_manager'],$res['id_currency'], 0,$res['MCC'],$res['MNC'],$res['id_name_delivery'],$date,$res['originator'],'not_deliver',0,$res['count_sms'],$res['sell_we'],$res['buy_we'],$res['sell_partner'],$res['buy_partner']);
            }
        }
    }

    function update_sms_stat($id_user,$id_partner,$id_manager,$id_currency,$id_currency_aggregating,$MCC,$MNC,$id_name_delivery,$date,$originator,$state,$id_aggregating,$count_sms,$sell_we,$buy_we,$sell_partner,$buy_partner,$repayment_sell_we=0,$repayment_buy_we=0,$repayment_sell_partner=0,$repayment_buy_partner=0)
    {
        global $SQL, $convertion_currency;
        $query = 'select id_stat from sms_stat where id_user="' . $id_user . '" and id_partner="' . $id_partner . '" and id_manager="' . $id_manager . '" and MCC="' . $MCC . '" and MNC="' . $MNC . '" and id_name_delivery="' . $id_name_delivery . '" and date="' . $date . '" and originator="' . $originator . '" and state="'.$state.'" and id_aggregating="'.$id_aggregating.'"';
        $result = $SQL->query($query);
        if($res = $SQL->fetch_assoc($result))
        {
            $query = 'update sms_stat set sum_parts=sum_parts+"' . $count_sms . '" where id_stat="' . $res['id_stat'] . '" ';
            $SQL->query_async($query);

            if(is_array($convertion_currency))
            {
                foreach($convertion_currency as $id_currency_need => $value)
                {
                    $query = 'update sms_stat_currency set sell_we=sell_we+"' . get_convertion_currency($sell_we, $id_currency, $id_currency_need) . '", buy_we=buy_we+"' . get_convertion_currency($buy_we, $id_currency_aggregating, $id_currency_need) . '", sell_partner=sell_partner+"' .get_convertion_currency($sell_partner, $id_currency, $id_currency_need) . '", buy_partner=buy_partner+"' . get_convertion_currency($buy_partner, $id_currency, $id_currency_need) . '",repayment_sell_we=repayment_sell_we+"' . get_convertion_currency($repayment_sell_we, $id_currency, $id_currency_need) . '", repayment_buy_we=repayment_buy_we+"' . get_convertion_currency($repayment_buy_we, $id_currency_aggregating, $id_currency_need) . '", repayment_sell_partner=repayment_sell_partner+"' .get_convertion_currency($repayment_sell_partner, $id_currency, $id_currency_need) . '",
                        repayment_buy_partner=repayment_buy_partner+"' . get_convertion_currency($repayment_buy_partner, $id_currency, $id_currency_need) . '" where id_stat="' . $res['id_stat'] . '" and id_currency="'.$id_currency_need.'"';
                    $SQL->query_async($query);
                }
            }
        }
        else
        {
            $query = 'insert into sms_stat set sum_parts="' . $count_sms . '", id_user="' . $id_user . '", id_partner="' . $id_partner . '", id_manager="' . $id_manager . '", MCC="' . $MCC . '", MNC="' . $MNC . '", id_name_delivery="' . $id_name_delivery . '", date="' . $date . '", originator="' . $originator . '", state="'.$state.'", id_aggregating="'.$id_aggregating.'"';
            $SQL->query($query);
            $id_stat = $SQL->insert_id();
            if(is_array($convertion_currency))
            {
                foreach($convertion_currency as $id_currency_need => $value)
                {
                    $query = 'insert into sms_stat_currency set sell_we=sell_we+"' . get_convertion_currency($sell_we, $id_currency, $id_currency_need) . '", buy_we=buy_we+"' . get_convertion_currency($buy_we, $id_currency_aggregating, $id_currency_need) . '", sell_partner=sell_partner+"' .get_convertion_currency($sell_partner, $id_currency, $id_currency_need) . '", buy_partner=buy_partner+"' . get_convertion_currency($buy_partner, $id_currency, $id_currency_need) . '",repayment_sell_we=repayment_sell_we+"' . get_convertion_currency($repayment_sell_we, $id_currency, $id_currency_need) . '", repayment_buy_we=repayment_buy_we+"' . get_convertion_currency($repayment_buy_we, $id_currency_aggregating, $id_currency_need) . '", repayment_sell_partner=repayment_sell_partner+"' .get_convertion_currency($repayment_sell_partner, $id_currency, $id_currency_need) . '",
                        repayment_buy_partner=repayment_buy_partner+"' . get_convertion_currency($repayment_buy_partner, $id_currency, $id_currency_need) . '", id_stat="' . $id_stat . '" , id_currency="'.$id_currency_need.'"';
                    $SQL->query_async($query);
                }
            }
        }
        $date = date('Y-m-d',(strtotime($date)+10800));
        $date = $date . ' 00:00:00';
        $query = 'select id_stat from bill_stat where id_user="' . $id_user . '" and id_partner="' . $id_partner . '" and id_manager="' . $id_manager . '" and MCC="' . $MCC . '" and MNC="' . $MNC . '" and id_name_delivery="' . $id_name_delivery . '" and date="' . $date . '" and originator="' . $originator . '" and state="'.$state.'" and id_aggregating="'.$id_aggregating.'"';
        $result = $SQL->query($query);
        if($res = $SQL->fetch_assoc($result))
        {
            $query = 'update bill_stat set sum_parts=sum_parts+"' . $count_sms . '" where id_stat="' . $res['id_stat'] . '" ';
            $SQL->query_async($query);

            if(is_array($convertion_currency))
            {
                foreach($convertion_currency as $id_currency_need => $value)
                {
                    $query = 'update bill_stat_currency set sell_we=sell_we+"' . get_convertion_currency($sell_we, $id_currency, $id_currency_need) . '", buy_we=buy_we+"' . get_convertion_currency($buy_we, $id_currency_aggregating, $id_currency_need) . '", sell_partner=sell_partner+"' .get_convertion_currency($sell_partner, $id_currency, $id_currency_need) . '", buy_partner=buy_partner+"' . get_convertion_currency($buy_partner, $id_currency, $id_currency_need) . '",repayment_sell_we=repayment_sell_we+"' . get_convertion_currency($repayment_sell_we, $id_currency, $id_currency_need) . '", repayment_buy_we=repayment_buy_we+"' . get_convertion_currency($repayment_buy_we, $id_currency_aggregating, $id_currency_need) . '", repayment_sell_partner=repayment_sell_partner+"' .get_convertion_currency($repayment_sell_partner, $id_currency, $id_currency_need) . '",
                        repayment_buy_partner=repayment_buy_partner+"' . get_convertion_currency($repayment_buy_partner, $id_currency, $id_currency_need) . '" where id_stat="' . $res['id_stat'] . '" and id_currency="'.$id_currency_need.'"';
                    $SQL->query_async($query);
                }
            }
        }
        else
        {
            $query = 'insert into bill_stat set sum_parts="' . $count_sms . '", id_user="' . $id_user . '", id_partner="' . $id_partner . '", id_manager="' . $id_manager . '", MCC="' . $MCC . '", MNC="' . $MNC . '", id_name_delivery="' . $id_name_delivery . '", date="' . $date . '", originator="' . $originator . '", state="'.$state.'", id_aggregating="'.$id_aggregating.'"';
            $SQL->query($query);
            $id_stat = $SQL->insert_id();
            if(is_array($convertion_currency))
            {
                foreach($convertion_currency as $id_currency_need => $value)
                {
                    $query = 'insert into bill_stat_currency set sell_we=sell_we+"' . get_convertion_currency($sell_we, $id_currency, $id_currency_need) . '", buy_we=buy_we+"' . get_convertion_currency($buy_we, $id_currency_aggregating, $id_currency_need) . '", sell_partner=sell_partner+"' .get_convertion_currency($sell_partner, $id_currency, $id_currency_need) . '", buy_partner=buy_partner+"' . get_convertion_currency($buy_partner, $id_currency, $id_currency_need) . '",repayment_sell_we=repayment_sell_we+"' . get_convertion_currency($repayment_sell_we, $id_currency, $id_currency_need) . '", repayment_buy_we=repayment_buy_we+"' . get_convertion_currency($repayment_buy_we, $id_currency_aggregating, $id_currency_need) . '", repayment_sell_partner=repayment_sell_partner+"' .get_convertion_currency($repayment_sell_partner, $id_currency, $id_currency_need) . '",
                        repayment_buy_partner=repayment_buy_partner+"' . get_convertion_currency($repayment_buy_partner, $id_currency, $id_currency_need) . '", id_stat="' . $id_stat . '" , id_currency="'.$id_currency_need.'"';
                    $SQL->query_async($query);
                }
            }
        }
    }

    function update_sms_stat_hlr($id_user,$id_partner,$id_manager,$id_currency,$id_currency_aggregating,$MCC,$MNC,$id_name_delivery,$date,$originator,$state,$id_aggregating,$count_sms)
    {
        global $SQL, $convertion_currency,$query_log;
        $query = 'select id_stat from sms_stat_hlr where id_user="' . $id_user . '" and id_partner="' . $id_partner . '" and id_manager="' . $id_manager . '" and MCC="' . $MCC . '" and MNC="' . $MNC . '" and id_name_delivery="' . $id_name_delivery . '" and date="' . $date . '" and originator="' . $originator . '" and state="'.$state.'" and id_aggregating_hlr="'.$id_aggregating.'"';
        $result = $SQL->query($query);
        if($res = $SQL->fetch_assoc($result))
        {
            $query = 'update sms_stat_hlr set sum_parts=sum_parts+"' . $count_sms . '" where id_stat="' . $res['id_stat'] . '" ';
            $SQL->query_async($query);
        }
        else
        {
            $query = 'insert into sms_stat_hlr set sum_parts="' . $count_sms . '", id_user="' . $id_user . '", id_partner="' . $id_partner . '", id_manager="' . $id_manager . '", MCC="' . $MCC . '", MNC="' . $MNC . '", id_name_delivery="' . $id_name_delivery . '", date="' . $date . '", originator="' . $originator . '", state="'.$state.'", id_aggregating_hlr="'.$id_aggregating.'"';
            $SQL->query($query);
        }
        $query_log = $query;
    }    
    
    function return_control_sms($id_control_text, $table)
    {
        global $SQL;
        $query = 'select id_turn, id_user, count_sms, id_package_user, sell_we, sell_partner, id_package_partner from '.$table.' where id_control_text="' . $id_control_text . '"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++)
        {
            $id_turn=$SQL->result($result, $i, 'id_turn');
            $id_user=$SQL->result($result, $i, 'id_user');
            if($id_user)
            {
                $user = get_param_id_user($id_user);
                if($user['pay_control'])
                {
                    sms_final_pay_control($id_turn);
                    continue;
                }
            }

            $sell_we = $SQL->result($result, $i, 'sell_we');
            $sell_partner = $SQL->result($result, $i, 'sell_partner');
            $id_package_user = $SQL->result($result, $i, 'id_package_user');
            $id_package_partner = $SQL->result($result, $i, 'id_package_partner');
            $count_sms = $SQL->result($result, $i, 'count_sms');

            if($id_package_partner)
            {
                $query = 'update sms_package_partner set waste=waste-'.$sell_we.' where id_package_partner="'.$id_package_partner.'"';
                $SQL->query($query);
                $sell_we=$sell_partner;
            }
            $query = 'update sms_package_user set waste=waste-'.$sell_we.' where id_package_user="'.$id_package_user.'"';
            $SQL->query($query);
            $query = 'delete from ' . $table . ' where id_turn="' . addslashes($id_turn) . '"';
            $SQL->query($query);
            $query = 'delete from sms_turn_HLR where id_turn="' . addslashes($id) . '" or id_turn="-' . addslashes($id) . '"';
            $SQL->query($query);
            $query = 'delete from sms_state_final where id_sms="-' . addslashes($id_turn) . '"';
            $SQL->query($query);
        }
    }

    function check_package($id_user, $id_partner, $date, $phone = '', $type_sms='sms', $count_send_sms = 0)
    {
        global $SQL;
        $sell_we = 0;
        $sell_partner = 0;
        $buy_partner = 0;
        $phone_code = get_phone_code_from_array($phone);
        $query = 'select pu.id_package_user, pu.test_sms, pu.id_currency, pu.limit_price, (pu.limit_price+pu.credit-pu.waste) as rest from sms_package_user as pu where pu.id_user="' . $id_user . '" and pu.date_start<="' . $date . '" and pu.date_stop>="' . $date . '" and ( pu.limit_price+pu.credit-pu.waste>0 or pu.limit_price=0) order by pu.date_start asc';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
        {
            if($count_send_sms)
            {
                $id_package_user=0;
                for($i=0; $i<$num; $i++)
                {
                    $id_package_user_ = $SQL->result($result, $i, 'pu.id_package_user');
                    $rest= $SQL->result($result, $i, 'rest');
                    $limit_price= $SQL->result($result, $i, 'pu.limit_price');
                    $query_price = 'select price_sms, MNC, deliver_payment from sms_price_user where id_package_user="' . $id_package_user_ . '" and ((MCC="' .$phone_code[ 'MCC'] . '" and MNC="' . $phone_code[ 'MNC'] . '") or (MCC="' . $phone_code[ 'MCC'] . '" and MNC="0")) and  type_sms="'.$type_sms.'" order by MNC desc limit 0,1';
                    $result_price = $SQL->query($query_price);
                    if($SQL->num_rows($result_price))
                    {
                        //$phone_code = (string)$SQL->result($result_price, $j, 'spc.phone_code');
                        $price_sms = $SQL->result($result_price,0, 'price_sms');
                        //$count_sms = $SQL->result($result_price, $j, 'spu.count_sms');
                        if($limit_price==0||$rest>=$price_sms*$count_send_sms)
                        {
                             $MNC_user = $SQL->result($result_price, 0, 'MNC');
                             $test_sms = $SQL->result($result, $i, 'pu.test_sms');
                             $price_sms_user=$price_sms;
                             $id_package_user = $id_package_user_;
                             $id_currency = $SQL->result($result, $i, 'pu.id_currency');
                             $deliver_payment_user = $SQL->result($result_price, 0, 'pu.deliver_payment');
                             break;
                        }
                    }
                }
            }
            else
            {
                $id_package_user = $SQL->result($result, 0, 'pu.id_package_user');
                $test_sms = $SQL->result($result, 0, 'pu.test_sms');
                $id_currency = $SQL->result($result, 0, 'pu.id_currency');
            }

            if($id_package_user)
            {
                $id_package_partner = 0;
                if($id_partner)
                {
                    $query = "select sql_cache block from sms_partners where id_partner='" . $id_partner . "'";
                    $result = $SQL->query($query);
                    if($SQL->num_rows($result))
                    {
                        $block = $SQL->result($result, 0, 'block');
                        if($block)
                            return array('error' => 'block_partner','MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC']);
                    }
                    else
                        return array('error' => 'block_partner','MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC']);

                    if(!$test_sms)
                    {
                        $query = 'select pu.id_package_partner, pu.limit_price, pu.limit_price+pu.credit-pu.waste as rest from sms_package_partner as pu where pu.id_partner="' . $id_partner . '" and pu.date_start<="' . $date . '" and pu.date_stop>="' . $date . '" and (pu.limit_price+pu.credit-pu.waste>0 or pu.limit_price=0) and id_currency="'.$id_currency.'" order by pu.date_start asc';
                        $price_send_sms=0;
                        $result = $SQL->query($query);
                        $num = $SQL->num_rows($result);
                        if($num)
                        {
                            if($count_send_sms)
                            {
                               $id_package_partner = 0;
                                for($i=0; $i<$num; $i++)
                                {
                                    $id_package_partner_ = $SQL->result($result, $i, 'pu.id_package_partner');
                                     $limit_price= $SQL->result($result, $i, 'pu.limit_price');
                                    $rest= $SQL->result($result, $i, 'rest');
                                    $query_price = 'select price_sms, MNC, deliver_payment from sms_price_partner where id_package_partner="' . $id_package_partner_ . '" and ((MCC="' .$phone_code[ 'MCC'] . '" and MNC="' . $phone_code[ 'MNC'] . '") or (MCC="' . $phone_code[ 'MCC'] . '" and MNC="0")) and type_sms="'.$type_sms.'" order by MNC desc limit 0,1';
                                    $result_price = $SQL->query($query_price);
                                     if($SQL->num_rows($result_price))
                                      {

                                          //$phone_code = (string)$SQL->result($result_price, $j, 'spc.phone_code');
                                          $price_sms = $SQL->result($result_price, 0, 'price_sms');
                                          //$count_sms = $SQL->result($result_price, $j, 'spu.count_sms');
                                          if($limit_price==0||$rest>=$price_sms*$count_send_sms)
                                          {
                                               $MNC_partner = $SQL->result($result_price, 0, 'MNC');
                                               $price_sms_partner=$price_sms;
                                               $id_package_partner = $id_package_partner_;
                                               $deliver_payment_partner = $SQL->result($result_price, 0, 'pu.deliver_payment');
                                               break;
                                          }
                                      }
                                }
                                    if($id_package_partner)
                                    {

                                        $query = 'update sms_package_partner set waste=waste+'.$price_sms_partner*$count_send_sms.' where id_package_partner="'.$id_package_partner.'"';
                                        $SQL->query_async($query);
                                    }
                                    else
                                       return array('error' => 'phone_code_partner','MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC']);
                            }
                            else
                                $id_package_partner = $SQL->result($result, 0, 'pu.id_package_partner');
                        }
                        else
                            return array('error' => 'package_partner','MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC']);
                    }
                }

                if($count_send_sms)
                {
                    $query = 'update sms_package_user set waste=waste+'.$price_sms_user*$count_send_sms.' where id_package_user="'.$id_package_user.'"';
                    $SQL->query_async($query);
                }
                return array('id_package_user' => $id_package_user, 'id_package_partner' => $id_package_partner,'MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC'], 'MNC_user' => $MNC_user, 'MNC_partner' => $MNC_partner, 'test_sms' => $test_sms, 'price_sms_user' => $price_sms_user, 'deliver_payment_user' => $deliver_payment_user, 'price_sms_partner' => $price_sms_partner, 'deliver_payment_partner' => $deliver_payment_partner, 'id_currency' => $id_currency);
            }
            else
                return array('error' => 'phone_code_user','MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC']);
        }
        else
            return array('error' => 'package_user','MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC']);
    }

    function check_phone_code($id_user, $MCC, $MNC)
    {
        global $SQL;
        $query = 'select spru.id_package_user from sms_package_user as spu, sms_price_user as spru WHERE spu.id_package_user=spru.id_package_user and spru.MCC="'.$MCC.'" and (spru.MNC="'.$MNC.'" or spru.MNC="0") and spu.id_user="' . $id_user . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
            return TRUE;
        else
            return FALSE;
    }

    function check_sync($id_user, $sync)
    {
        global $SQL;
        $id_state = FALSE;
        if($sync)
        {
            $query = 'select id_state from sms_state_final where id_user="' . $id_user . '" and sync="' . $sync . '"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
                $id_state = $SQL->result($result, 0, 'id_state');
        }
        return $id_state;
    }

    function get_control_text($text_sms)
    {
        $text_sms = trim($text_sms);
        $len = strlen($text_sms);
        $have_space = 1;
        for($i=0; $i<$len; $i++)
        {
            if($text_sms[$i] != ' ' && $text_sms[$i] != chr(10) && $text_sms[$i] != chr(13))
            {
                $text .= $text_sms[$i];
                $have_space = 0;
            }
            elseif($have_space == 0)
            {
                $text .= '|';
                $have_space = 1;
            }
        }
        if(substr($text, -1, 1) == '|')
            $text = substr($text, 0, -1);
        return $text;
    }

    function send_three_sms($auth_user, $all_originator, $control_text_user, $id_partner, $phone, $originator, $flash_sms, $text_sms, $text_wap, $url, $name, $phone_cell, $phone_work, $phone_fax, $e_mail, $url_vcard, $position, $organization, $address_post_office_box, $address_street, $address_city, $address_region, $address_postal_code, $address_country, $additional, $date_wait, $from, $name_delivery, $vp_hour, $vp_minute, $priority)
    {
        global $SQL, $global_set_message_wait;

        $date = date('Y-m-d');
        $hour = date('H:i');
        $hour = explode(':', $hour);
        $send_from = explode(':', $from);
        if(strcmp($date_wait, $date) > 0 || (strcmp($date_wait, $date) == 0 && ($send_from[0] > $hour[0] || ($send_from[0] == $hour[0] && $send_from[1] > $hour[1]))))
            $validity_period = date_add_minute($date_wait . ' ' . $from . ':00', $vp_hour*60+$vp_minute);
        else
            $validity_period = date('Y-m-d H:i:s', (time()+($vp_hour*60 + $vp_minute)*60));
        if($text_sms)
        {
            if($flash_sms)
                $type = 'flashsms';
            else
                $type = 'sms';
            $send_sms[0] = send_sms($auth_user, $all_originator, $control_text_user, $id_partner, $phone, $originator, $type, $text_sms, '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', $date_wait, $from, $name_delivery, 'all', 0, 0, 0, $validity_period, 1, $priority);
        }
        if($url)
        {
            $type = 'wappush';
            $send_sms[1] = send_sms($auth_user, $all_originator, $control_text_user, $id_partner, $phone, $originator, 'wappush', $text_wap, $url, '', '', '', '', '', '', '', '', '', '', '', '', '', '', $date_wait, $from, $name_delivery, 'all', 0, 0, 0, $validity_period, 1, $priority);
        }
        if($name)
        {
            $type = 'vcard';
            $send_sms[2] = send_sms($auth_user, $all_originator, $control_text_user, $id_partner, $phone, $originator, 'vcard', '', $url_vcard, $name, $phone_cell, $phone_work, $phone_fax, $e_mail, $position, $organization, $address_post_office_box, $address_street, $address_city, $address_region, $address_postal_code, $address_country, $additional, $date_wait, $from, $name_delivery, 'all', 0, 0, 0, $validity_period, 1, $priority);
        }
        if(is_array($send_sms))
        {
           foreach($send_sms as $key => $value)
           {
               if(($value['error'] == 'phone_code_partner' || $value['error'] == 'package_partner' || $value['error'] == 'phone_code_user' || $value['error'] == 'package_user') && check_phone_code($auth_user, $value['MCC'], $value['MNC']))
               {
                   $global_set_message_wait += $value['count_sms'];
                   $send_sms[$key]['error'] = 'sms_wait_payment';
                   if($type == 'wappush')
                       $text_sms_ = set_param_sms(array($url, $text_sms));
                   elseif($type == 'vcard')
                       $text_sms_ = set_param_sms(array($name, $phone_cell, $phone_work, $phone_fax, $e_mail, $position, $organization, $url, $address_post_office_box, $address_street, $address_city, $address_region, $address_postal_code, $address_country, $additional));
                   else
                       $text_sms_ = $text_sms;
                   $query = 'insert into sms_wait set id_user="' . $auth_user . '", originator="' . $originator . '", type_sms="' . $type . '", text_sms="' . $text_sms_ . '", phone="' . $phone . '", send_from="' . $from . ':00", send_from_date="' . $date_wait . '", name_delivery="' . $name_delivery . '", validity_period="' . $validity_period . '"';
                   $SQL->query($query);
               }
           }
           return $send_sms;
        }
        return FALSE;
    }

    function set_control()
    {
        global $sms_control_text_control;
        unset($sms_control_text_control);
        global $SQL, $sms_control_text_control;
        $sms_control_text_control = array();
        $query = 'select sql_cache control, where_search from sms_automatic_control';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++)
        {
            $control = $SQL->result($result, $i, 'control');
            $where_search = $SQL->result($result, $i, 'where_search');
            $sms_control_text_control[] = array('control' => $control, 'where_search' => $where_search);
        }
    }

    function check_control($originator, $text_sms)
    {
        global $sms_control_text_control;
        if(is_array($sms_control_text_control))
        {
            foreach($sms_control_text_control as $control)
            {
                if(($control['where_search'] == 'all' || $control['where_search'] == 'text') && preg_match('/' . $control['control'] . '/i', $text_sms))
                    return TRUE;
                if(($control['where_search'] == 'all' || $control['where_search'] == 'originator') && preg_match('/' . $control['control'] . '/i', $originator))
                    return TRUE;
            }
        }
        return FALSE;
    }
    function check_phone($phone = ''){
        if($phone!=''){
            $phone_code = get_phone_code_from_array($phone);
            if($phone_code['MCC']>0){
                $result = TRUE;
            }else{
                $result = FALSE;
            }
        }else{
            $result = FALSE;
        }
        return $result;
    }
    
    function check_package_no_change($id_user, $id_partner, $date, $phone = '', $type_sms='sms', $count_send_sms = 1)
    {
        global $SQL;
        $sell_we = 0;
        $sell_partner = 0;
        $buy_partner = 0;
        $phone_code = get_phone_code_from_array($phone);
        $query = 'select pu.id_package_user, pu.test_sms, pu.id_currency, pu.limit_price, (pu.limit_price+pu.credit-pu.waste) as rest from sms_package_user as pu where pu.id_user="' . $id_user . '" and pu.date_start<="' . $date . '" and pu.date_stop>="' . $date . '" and ( pu.limit_price+pu.credit-pu.waste>0 or pu.limit_price=0) order by pu.date_start asc';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
        {
            if($count_send_sms)
            {
               $id_package_user=0;
                for($i=0; $i<$num; $i++)
                {
                    $id_package_user_ = $SQL->result($result, $i, 'pu.id_package_user');
                    $rest= $SQL->result($result, $i, 'rest');
                    $limit_price= $SQL->result($result, $i, 'pu.limit_price');
                    $query_price = 'select price_sms, MNC, deliver_payment from sms_price_user where id_package_user="' . $id_package_user_ . '" and ((MCC="' .$phone_code[ 'MCC'] . '" and MNC="' . $phone_code[ 'MNC'] . '") or (MCC="' . $phone_code[ 'MCC'] . '" and MNC="0")) and  type_sms="'.$type_sms.'" order by MNC desc limit 0,1';
                    $result_price = $SQL->query($query_price);
                    if($SQL->num_rows($result_price))
                    {
                        //$phone_code = (string)$SQL->result($result_price, $j, 'spc.phone_code');
                        $price_sms = $SQL->result($result_price,0, 'price_sms');
                        //$count_sms = $SQL->result($result_price, $j, 'spu.count_sms');
                        if($limit_price==0||$rest>=$price_sms*$count_send_sms)
                        {
                             $MNC_user = $SQL->result($result_price, 0, 'MNC');
                             $test_sms = $SQL->result($result, $i, 'pu.test_sms');
                             $price_sms_user=$price_sms;
                             $id_package_user = $id_package_user_;
                             $id_currency = $SQL->result($result, $i, 'pu.id_currency');
                             $deliver_payment_user = $SQL->result($result_price, 0, 'pu.deliver_payment');
                             break;
                        }
                    }
                }
            }
            else
            {
                $id_package_user = $SQL->result($result, 0, 'pu.id_package_user');
                $test_sms = $SQL->result($result, 0, 'pu.test_sms');
                $id_currency = $SQL->result($result, 0, 'pu.id_currency');
            }

            if($id_package_user)
            {
                $id_package_partner = 0;
                if($id_partner)
                {
                    $query = "select sql_cache block from sms_partners where id_partner='" . $id_partner . "'";
                    $result = $SQL->query($query);
                    if($SQL->num_rows($result))
                    {
                        $block = $SQL->result($result, 0, 'block');
                        if($block)
                            return array('error' => 'block_partner','MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC']);
                    }
                    else
                        return array('error' => 'block_partner','MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC']);

                    if(!$test_sms)
                    {
                        $query = 'select pu.id_package_partner, pu.limit_price, pu.limit_price+pu.credit-pu.waste as rest from sms_package_partner as pu where pu.id_partner="' . $id_partner . '" and pu.date_start<="' . $date . '" and pu.date_stop>="' . $date . '" and (pu.limit_price+pu.credit-pu.waste>0 or pu.limit_price=0) and id_currency="'.$id_currency.'" order by pu.date_start asc';
                        $price_send_sms=0;
                        $result = $SQL->query($query);
                        $num = $SQL->num_rows($result);
                        if($num)
                        {
                            if($count_send_sms)
                            {
                               $id_package_partner = 0;
                                for($i=0; $i<$num; $i++)
                                {
                                    $id_package_partner_ = $SQL->result($result, $i, 'pu.id_package_partner');
                                     $limit_price= $SQL->result($result, $i, 'pu.limit_price');
                                    $rest= $SQL->result($result, $i, 'rest');
                                    $query_price = 'select price_sms, MNC, deliver_payment from sms_price_partner where id_package_partner="' . $id_package_partner_ . '" and ((MCC="' .$phone_code[ 'MCC'] . '" and MNC="' . $phone_code[ 'MNC'] . '") or (MCC="' . $phone_code[ 'MCC'] . '" and MNC="0")) and type_sms="'.$type_sms.'" order by MNC desc limit 0,1';
                                    $result_price = $SQL->query($query_price);
                                     if($SQL->num_rows($result_price))
                                      {

                                          //$phone_code = (string)$SQL->result($result_price, $j, 'spc.phone_code');
                                          $price_sms = $SQL->result($result_price, 0, 'price_sms');
                                          //$count_sms = $SQL->result($result_price, $j, 'spu.count_sms');
                                          if($limit_price==0||$rest>=$price_sms*$count_send_sms)
                                          {
                                               $MNC_partner = $SQL->result($result_price, 0, 'MNC');
                                               $price_sms_partner=$price_sms;
                                               $id_package_partner = $id_package_partner_;
                                               $deliver_payment_partner = $SQL->result($result_price, 0, 'pu.deliver_payment');
                                               break;
                                          }
                                      }
                                }
                                    if($id_package_partner)
                                    {

                                        return array('no_error' => 'no_error');
                                    }
                                    else
                                       return array('error' => 'phone_code_partner','MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC']);
                            }
                            else
                                $id_package_partner = $SQL->result($result, 0, 'pu.id_package_partner');
                        }
                        else
                            return array('error' => 'package_partner','MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC']);
                    }
                }

                if($count_send_sms)
                {
                    return array('no_error' => 'no_error');
                }
                return array('id_package_user' => $id_package_user, 'id_package_partner' => $id_package_partner,'MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC'], 'MNC_user' => $MNC_user, 'MNC_partner' => $MNC_partner, 'test_sms' => $test_sms, 'price_sms_user' => $price_sms_user, 'deliver_payment_user' => $deliver_payment_user, 'price_sms_partner' => $price_sms_partner, 'deliver_payment_partner' => $deliver_payment_partner, 'id_currency' => $id_currency);
            }
            else
                return array('error' => 'phone_code_user','MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC']);
        }
        else
            return array('error' => 'package_user','MCC' => $phone_code[ 'MCC'],'MNC'=>$phone_code[ 'MNC']);
    }
    // проверка части сообщения
    // $phone - номер в формате 79035284830
    // $text - текст сообщения
    function check_part($auth_user, $all_originator, $control_text_user, $id_partner, $phone, $originator, $type, $text_sms, $url = '', $name = '', $phone_cell = '', $phone_work = '', $phone_fax = '', $e_mail = '', $position = '', $organization = '', $address_post_office_box = '', $address_street = '', $address_city = '', $address_region = '', $address_postal_code = '', $address_country = '', $additional = '', $date_wait = '0000-00-00', $from = '00:00', $name_delivery = '', $insert_state = 'all', $sync = 0, $send = 0, $send_xml = 0, $validity_period = '', $check_global_stop_list = 1, $priority = 1000, $replace_phone = TRUE)
    {
        global $SQL;
        if($replace_phone) // Заменить телефонный номер? (в случае когда берёться номер из базы замены не происходит)
            $phone = get_replace_phone($phone);
        else
            $phone = get_phone($phone);

        if(check_phone($phone)) // Проверка если номер телефона на который отравлять
        {
           $alfanum = preg_match('/\D/', $originator);
           $len_orig = strlen(stripslashes($originator));
           if($len_orig==0)
               return array('error' => 'no_originator');
           if(($alfanum && $len_orig>11) || (!$alfanum && $len_orig>15))
               return array('error' => 'incorrect_originator');
           if(strlen($phone) > 15)
               return array('error' => 'incorrect_phone');

           if(($type == 'sms' || $type == 'flashsms') && $text_sms==='')
               return array('error' => 'no_text_sms');
           if($type == 'wappush' && !$url)
               return array('error' => 'no_url');
           if($type == 'vcard' && !($name && ($phone_cell || $phone_work || $phone_fax || $e_mail || $position || $organization || $url || $address_post_office_box || $address_street || $address_city || $address_region || $address_postal_code || $address_country || $additional)))
               return array('error' => 'no_vcard');

           if($all_originator == 0)
           {
               $query = 'select id_originator, originator, state from sms_originator where id_user="' . $auth_user . '" and originator="' . $originator . '"';
               $result = $SQL->query($query);
               if($SQL->num_rows($result) == 0)
                   return array('error' => 'not_registr_originator');
               elseif($SQL->result($result, 0, 'state') != 'completed')
                   return array('error' => 'not_control_originator');
           }

           if($type == 'wappush')
               $text_sms = set_param_sms(array($url, $text_sms));
           elseif($type == 'vcard')
               $text_sms = set_param_sms(array($name, $phone_cell, $phone_work, $phone_fax, $e_mail, $position, $organization, $url, $address_post_office_box, $address_street, $address_city, $address_region, $address_postal_code, $address_country, $additional));

           $text_sms = get_text_sms($text_sms);
           $count_sms = get_count_sms($type, $text_sms);
            if($check_global_stop_list)
            {
                $query = 'select phone from sms_global_stop where phone="'.$phone.'"';
                $result = $SQL->query($query);
                $have_global = $SQL->num_rows($result);
            }
            else
                $have_global = 0;
            if($have_global == 0)
            {
            $query = 'select phone from sms_stop where id_user="' . $auth_user . '" and phone="'.$phone.'"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result) == 0)
            {
                $date = date('Y-m-d');
                $check_package = check_package_no_change($auth_user, $id_partner, $date, $phone, $type, 1);// проверка пакета
                
                if(isset($check_package['error']))
                    return array('error' => $check_package['error']);
                else
                {
                   return array('no_error' => 'no_error');
                }
            }
            else
                return array('error' => 'stop_phone');
            }
            else
                return array('error' => 'stop_phone');
        }
        else
            return array('error' => 'no_phone');
    }
    /**
     * отправка SMS сообщения
     * @global type $SQL
     * @param int $auth_user id_user в системе
     * @param int $all_originator любое имя отправителя (0 или 1)
     * @param int $control_text_user 2 - автоматическая модерация, 1 - ручная, 0 -нет
     * @param int $id_partner  id партнера в системе
     * @param string $phone  номер в формате 79035284830
     * @param string $originator подпись отправиеля
     * @param string $type тип сообщения sms|flashsms|wappush|vcard
     * @param string $text_sms  текст сообщения
     * @param string $url  url использщуется только в wappush
     * @param string $name ФИО используется в VCARD
     * @param string $phone_cell сотовый телефон, используется в VCARD
     * @param string $phone_work рабочий телефон, используется в VCARD
     * @param string $phone_fax телефон для факсов, используется в VCARD
     * @param string $e_mail  адрес электронной почты, используется в VCARD
     * @param string $position это GEO метка или должность, используется в VCARD
     * @param string $organization название организации, используется в VCARD
     * @param string $address_post_office_box адрес почтового ящика офиса, используется в VCARD
     * @param string $address_street улица, используется в VCARD
     * @param string $address_city  город, используется в VCARD
     * @param string $address_region  регион, используется в VCARD
     * @param string $address_postal_code  post code, используется в VCARD
     * @param string $address_country  страна, используется в VCARD
     * @param string $additional произвольные поля VCARD, типа даа смерти, дата рождения, хобби и т.д.
     * @param string $date_wait дата до которой надо отправить (относится к постановке sms в очередь)
     * @param string $from по какое время отправлять (относится к постановке sms в очередь)
     * @param string $name_delivery имя рассылки
     * @param string $insert_state тех. параметр, для xml вегда должен быть all
     * @param int $sync  уникальный модификатор полученный от клиента
     * @param int $send отправлять отчет по smpp
     * @param int $send_xml отправленно по xml
     * @param string $validity_period валидно до
     * @param int $check_global_stop_list проверять в глобально стоп листе
     * @param int $priority приоритет
     * @param bool $replace_phone подменять ли 8 в номере
     * @return array
     */
    function send_sms($auth_user, $all_originator, $control_text_user, $id_partner, $phone, $originator, $type, $text_sms, $url = '', $name = '', $phone_cell = '', $phone_work = '', $phone_fax = '', $e_mail = '', $position = '', $organization = '', $address_post_office_box = '', $address_street = '', $address_city = '', $address_region = '', $address_postal_code = '', $address_country = '', $additional = '', $date_wait = '0000-00-00', $from = '00:00', $name_delivery = '', $insert_state = 'all', $sync = 0, $send = 0, $send_xml = 0, $validity_period = '', $check_global_stop_list = 1, $priority = 1000, $replace_phone = TRUE)
    {
        global $SQL;
        if($replace_phone) // Заменить телефонный номер? (в случае когда берёться номер из базы замены не происходит)
            $phone = get_replace_phone($phone);
        else
            $phone = get_phone($phone);

        if(check_phone($phone)) // Проверка если номер телефона на который отравлять
        {
           if(!check_originator($originator))
               return array('error' => 'incorrect_originator');
           if(strlen($phone) > 15)
               return array('error' => 'incorrect_phone');

           if(($type == 'sms' || $type == 'flashsms') && $text_sms==='')
               return array('error' => 'no_text_sms');
           if($type == 'wappush' && !$url)
               return array('error' => 'no_url');
           if($type == 'vcard' && !($name && ($phone_cell || $phone_work || $phone_fax || $e_mail || $position || $organization || $url || $address_post_office_box || $address_street || $address_city || $address_region || $address_postal_code || $address_country || $additional)))
               return array('error' => 'no_vcard');

           if($all_originator == 0)
           {
               $query = 'select id_originator, originator, state from sms_originator where id_user="' . $auth_user . '" and originator="' . addslashes($originator) . '"';
               $result = $SQL->query($query);
               if($SQL->num_rows($result) == 0)
                   return array('error' => 'not_registr_originator');
               elseif($SQL->result($result, 0, 'state') != 'completed')
                   return array('error' => 'not_control_originator');
           }

           if($type == 'wappush')
               $text_sms = set_param_sms(array($url, $text_sms));
           elseif($type == 'vcard')
               $text_sms = set_param_sms(array($name, $phone_cell, $phone_work, $phone_fax, $e_mail, $position, $organization, $url, $address_post_office_box, $address_street, $address_city, $address_region, $address_postal_code, $address_country, $additional));

           $text_sms = get_text_sms($text_sms);
           $count_sms = get_count_sms($type, $text_sms);
           if($id_state = check_sync($auth_user, $sync)) // Проверка по переменной $sync - отправлялось ли такое сообщение [?]
               return array('count_sms' => $count_sms, 'id_state' => $id_state, 'error' => 'already_send');
            if($check_global_stop_list)
            {
                $query = 'select phone from sms_global_stop where phone="'.$phone.'"';
                $result = $SQL->query($query);
                $have_global = $SQL->num_rows($result);
            }
            else
                $have_global = 0;
            if($have_global == 0)
            {
            $query = 'select phone from sms_stop where id_user="' . $auth_user . '" and phone="'.$phone.'"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result) == 0)
            {
                $date = date('Y-m-d');
                $check_package = check_package($auth_user, $id_partner, $date, $phone, $type, $count_sms);// Списываю деньги
                if($originator_=get_originator_from_array($auth_user,$check_package['MCC'],$check_package['MNC']))// Принудительного отправителя
                     $originator=$originator_;// Принудительного отправителя
                $prices=get_sell($check_package['price_sms_user'], $check_package['deliver_payment_user'], $check_package['test_sms'],$check_package['id_package_partner'],$check_package['price_sms_partner'], $check_package['deliver_payment_partner'],1);
                if(isset($check_package['error']))
                    return array('error' => $check_package['error'], 'count_sms' => $count_sms,'MCC' => $check_package['MCC'], 'MNC' => $check_package['MNC']);
                else
                {
                    $id_aggregating = get_id_aggregating($auth_user, $id_partner, $check_package['MCC'],$check_package['MNC'], $type, $originator, $count_sms);

                    if($check_package['test_sms'])
                        $control_text_user=2;
                    if($control_text_user)
                    {
                        if($control_text_user == 2)
                        {
                            if(check_control($originator, $text_sms)){
                                $control_text = get_control_text($originator . ' ' . $text_sms);
                                $query = 'select sql_cache id_control_text, state_control from sms_control_text where id_user="' . $auth_user . '" and control_text="' . $control_text . '" and id_aggregating="'.$id_aggregating.'" and state_control="order"';
                                $result = $SQL->query($query);
                                if($SQL->num_rows($result))
                                {
                                    $id_control_text = $SQL->result($result, 0, 'id_control_text');
                                }
                                else
                                {
                                    $query = 'insert into sms_control_text set id_user="' . $auth_user . '", control_text="' . $control_text . '", id_aggregating="'.$id_aggregating.'"';
                                    $SQL->query($query);
                                    $id_control_text = $SQL->insert_id();
                                    send_mail_control($auth_user);
                                }
                            } else
                                $id_control_text = 0;
                        } 
                        elseif($control_text_user == 1)
                        {
                            $control_text = get_control_text($originator . ' ' . $text_sms);
                            $query = 'select sql_cache id_control_text, state_control from sms_control_text where id_user="' . $auth_user . '" and control_text="' . $control_text . '" and id_aggregating="'.$id_aggregating.'"';
                            $result = $SQL->query($query);
                            if($SQL->num_rows($result))
                            {
                                $state_control = $SQL->result($result, 0, 'state_control');
                                if($state_control == 'completed')
                                    $id_control_text = 0;// Одобрен
                                elseif($state_control == 'rejected')
                                    return array('error' => 'control_text');
                                else
                                    $id_control_text = $SQL->result($result, 0, 'id_control_text');
                            }
                            else
                            {
                                $query = 'insert into sms_control_text set id_user="' . $auth_user . '", control_text="' . $control_text . '", id_aggregating="'.$id_aggregating.'"';
                                $SQL->query($query);
                                $id_control_text = $SQL->insert_id();
                                send_mail_control($auth_user);
                            }
                        }
                    }
                    else
                        $id_control_text = 0;

                    $id_name_delivery = get_id_name_delivery($name_delivery, $auth_user);
                    $id_turn = set_sms_turn($phone, $date_wait, $from, $auth_user, $id_control_text, $check_package['id_package_user'], $id_partner, $check_package['id_package_partner'], $check_package['MCC'], $check_package['MNC'],$check_package['MNC_user'],$check_package['MNC_partner'], $originator, $type, $text_sms, $count_sms, $check_package['id_currency'],$prices['sell_we']*$count_sms, $prices['sell_partner']*$count_sms , $prices['buy_partner']*$count_sms , $id_aggregating, $priority, $validity_period, $id_name_delivery);
                    if($insert_state == 'all') //
                    {
                        for($i=1; $i<=$count_sms; $i++)
                        {
                            if($i == 1)
                                $send_xml_ = $send_xml;
                            else
                                $send_xml_ = 0;
                            // [sms_state_final] Вставить стоимость Тут Списанную
                            $query = 'insert into sms_state_final set sell_we="' . $prices['sell_we'] . '", sell_partner="' . $prices['sell_partner'] . '", buy_partner="' . $prices['buy_partner'] . '", deliver_payment_sell_we="' . $prices['deliver_payment_sell_we'] . '", deliver_payment_sell_partner="' . $prices['deliver_payment_sell_partner'] . '", deliver_payment_buy_partner="' . $prices['deliver_payment_buy_partner'] . '", id_sms="' . $id_turn . '", part_no="' . $i . '", id_user="' . $auth_user . '", sync="' . $sync . '", send="0", send_xml="' . $send_xml_ . '", id_currency="'.$check_package['id_currency'].'"';
                            if($i == 1)
                            {
                                $SQL->query($query);
                                $id_state = $SQL->insert_id();
                            }
                            else
                                $SQL->query_async($query);
                        }
                    }
                    else // SMPP шлюз
                    {
                       // Вставить стоимость Тут Списанную
                        $query = 'insert into sms_state_final set sell_we="' . $prices['sell_we'] . '", sell_partner="' . $prices['sell_partner'] . '", buy_partner="' . $prices['buy_partner'] . '", deliver_payment_sell_we="' . $prices['deliver_payment_sell_we'] . '", deliver_payment_sell_partner="' . $prices['deliver_payment_sell_partner'] . '", deliver_payment_buy_partner="' . $prices['deliver_payment_buy_partner'] . '", id_sms="' . $id_turn . '", part_no="' . $count_sms . '", id_user="' . $auth_user . '", sync="' . $sync . '", send="' . $send . '", id_currency="'.$check_package['id_currency'].'"';
                        $SQL->query($query);
                        $id_state = $SQL->insert_id();
                    }

                    if($id_turn > 0)
                        $turn_time = FALSE;
                    else
                        $turn_time = TRUE;
                    return array('count_sms' => $count_sms, 'id_state' => $id_state, 'id_turn' => $id_turn, 'id_control_text' => $id_control_text, 'turn_time' => $turn_time, 'id_currency' => $check_package['id_currency'], 'sell_we'=>$prices['sell_we'], 'sell_partner'=>$prices['sell_partner'], 'buy_partner'=>$prices['buy_partner'], 'deliver_payment_sell_we' => $prices['deliver_payment_sell_we'], 'deliver_payment_buy_partner' => $prices['deliver_payment_buy_partner'], 'deliver_payment_sell_partner' => $prices['deliver_payment_sell_partner'], 'id_aggregating'=>$id_aggregating); // Вернуть стоимость Тут Списанную
                }
            }
            else
                return array('error' => 'stop_phone');
            }
            else
                return array('error' => 'stop_phone');
        } else {
            return array('error' => 'no_phone');
        }
    }

    function get_sell($price_sms_user, $deliver_payment_user, $test_sms_user, $id_package_partner, $price_sms_partner, $deliver_payment_partner, $count_sms)
    {
        if($test_sms_user == 0)
        {
            if($id_package_partner)
            {
               $buy_partner = $sell_we = $count_sms*$price_sms_partner;
               $sell_partner = $count_sms*$price_sms_user;
               $deliver_payment_buy_partner = $deliver_payment_sell_we = $deliver_payment_partner;
               $deliver_payment_sell_partner = $deliver_payment_user;
            }
            else
            {
               $sell_partner = $buy_partner = 0;
               $sell_we = $count_sms*$price_sms_user;
               $deliver_payment_buy_partner = $deliver_payment_sell_partner = 0;
               $deliver_payment_sell_we = $deliver_payment_user;
            }
        }
        else
        {
            $sell_we = 0;
            $sell_partner = 0;
            $buy_partner = 0;
            $deliver_payment_sell_we = $deliver_payment_buy_partner = $deliver_payment_sell_partner = 0;
        }
        return array('sell_we' => $sell_we, 'sell_partner' => $sell_partner, 'buy_partner' => $buy_partner, 'deliver_payment_sell_we' => $deliver_payment_sell_we, 'deliver_payment_buy_partner' => $deliver_payment_buy_partner, 'deliver_payment_sell_partner' => $deliver_payment_sell_partner);
    }
    
/**
 * Агрегация данных по кол-ву отправкок hlr запросов
 * @param type $id_user
 * @param type $id_partner
 * @param type $MCC
 * @param type $MNC
 * @param type $id_name_delivery
 * @param type $originator
 * @param type $state
 * @param type $id_aggregating
 * @param type $count_sms
 * @param type $id_currency
 * @param type $id_currency_aggregating
 * @param type $sell_we
 * @param type $buy_we
 * @param type $sell_partner
 * @param type $buy_partner
 * @param type $repayment_sell_we
 * @param type $repayment_buy_we
 * @param type $repayment_sell_partner
 * @param type $repayment_buy_partner
 * @param type $date
 * @param type $replace_partly_deliver
 * @return type
 */
    function set_sms_stat_hlr($id_user, $id_partner, $MCC, $MNC, $id_name_delivery, $originator, $state, $id_aggregating, $count_sms, $date, $replace_partly_deliver = 1)
    {
        global $query_log;
        $originator = addslashes($originator);
        $param = get_param_id_user($id_user);

        update_sms_stat_hlr($id_user,$id_partner,$param['id_manager'],$id_currency, $id_currency_aggregating, $MCC,$MNC,$id_name_delivery,$date,$originator,$state,$id_aggregating,$count_sms);
        if($replace_partly_deliver)
            update_sms_stat_hlr($id_user,$id_partner,$param['id_manager'],$id_currency, $id_currency_aggregating, $MCC,$MNC,$id_name_delivery,$date,$originator,'partly_deliver',$id_aggregating,-$count_sms);
        return $prices;
    }

// [set_sms_stat] Заполнение общей статистики
    function set_sms_stat($id_user, $id_partner, $MCC, $MNC, $id_name_delivery, $originator, $state, $id_aggregating, $count_sms, $id_currency, $id_currency_aggregating, $sell_we , $buy_we , $sell_partner , $buy_partner, $repayment_sell_we, $repayment_buy_we, $repayment_sell_partner, $repayment_buy_partner,  $date, $replace_partly_deliver = 1)
    {
        $originator = addslashes($originator);
        $param = get_param_id_user($id_user);

        update_sms_stat($id_user,$id_partner,$param['id_manager'],$id_currency, $id_currency_aggregating, $MCC,$MNC,$id_name_delivery,$date,$originator,$state,$id_aggregating,$count_sms,$sell_we,$buy_we,$sell_partner,$buy_partner,$repayment_sell_we,$repayment_buy_we,$repayment_sell_partner,$repayment_buy_partner);
        if($replace_partly_deliver)
            update_sms_stat($id_user,$id_partner,$param['id_manager'],$id_currency, $id_currency_aggregating, $MCC,$MNC,$id_name_delivery,$date,$originator,'partly_deliver',$id_aggregating,-$count_sms,-$sell_we,-$buy_we,-$sell_partner,-$buy_partner);
        return $prices;
    }

    function get_param_package($id_package,$type,$MCC=0, $MNC=0, $type_sms='sms')
    {
        global $SQL, $GLOBAL_CACHE_DATA;
        if(isset($GLOBAL_CACHE_DATA['get_param_package'][$type][$id_package][$MCC][$MNC][$type_sms]))
            return $GLOBAL_CACHE_DATA['get_param_package'][$type][$id_package][$MCC][$MNC][$type_sms];
        else
        {
            $query = 'select * from sms_package_' . $type . ' where id_package_' . $type . '="' . $id_package . '"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
            {
                if($type == 'user')
                    $GLOBAL_CACHE_DATA['get_param_package'][$type][$id_package][$MCC][$MNC][$type_sms]['test_sms'] = $SQL->result($result, 0, 'test_sms');
                if($MCC)
                   {
                         $query = 'select price_sms from sms_price_' . $type . ' where id_package_' . $type . '="' . $id_package . '" and MCC="' . $MCC . '" and MNC="' . $MNC . '"';
                         $result = $SQL->query($query);
                         if($SQL->num_rows($result))
                         {
                            $GLOBAL_CACHE_DATA['get_param_package'][$type][$id_package][$MCC][$MNC][$type_sms]['price_sms'] = $SQL->result($result, 0, 'price_sms');
                            $GLOBAL_CACHE_DATA['get_param_package'][$type][$id_package][$MCC][$MNC][$type_sms]['deliver_payment'] = $SQL->result($result, 0, 'deliver_payment');

                         }
                         else
                                return FALSE;
                   }
                   return $GLOBAL_CACHE_DATA['get_param_package'][$type][$id_package][$MCC][$MNC][$type_sms];
            }
            else
                return FALSE;
        }
    }

    function get_error($send_sms)
    {
       global $global_languages;
        if(empty($send_sms['error']))
            return FALSE;
        elseif($send_sms['error'] == 'package_partner')
            return array('fatal_error' => $global_languages['END_SMS_MANAGER']);
        elseif($send_sms['error'] == 'package_user')
            return array('fatal_error' => $global_languages['END_SMS']);
        elseif($send_sms['error'] == 'block_partner')
            return array('fatal_error' => $global_languages['ACCOUNT_BLOCKED']);
        elseif($send_sms['error'] == 'no_phone')
            return array('error' => $global_languages['INPUT_NUMBER_PHONE']);
        elseif($send_sms['error'] == 'stop_phone')
            return array('error' => $global_languages['NUMBER_PHONE_IN_STOP_LIST']);
        elseif($send_sms['error'] == 'phone_code_user')
            return array('error' => $global_languages['DIRECTION_CLOSE_FOR_YOU']);
        elseif($send_sms['error'] == 'phone_code_partner')
            return array('error' => $global_languages['DIRECTION_CLOSE']);
        elseif($send_sms['error'] == 'sms_wait_payment')
            return array('error' => $global_languages['NOT_ENOUGH_MONEY_SEND_SMS']);
        elseif($send_sms['error'] == 'control_text')
            return array('fatal_error' => $global_languages['TEXT_SMS_DENIED_MODERATOR']);
        elseif($send_sms['error'] == 'no_originator')
            return array('fatal_error' => $global_languages['NO_ORIGINATOR']);
        elseif($send_sms['error'] == 'incorrect_originator')
            return array('fatal_error' => $global_languages['INCORRECT_ORIGINATOR']);
        elseif($send_sms['error'] == 'incorrect_phone')
            return array('error' => $global_languages['INCORRECT_PHONE']);
        elseif($send_sms['error'] == 'no_text_sms')
            return array('fatal_error' => $global_languages['NO_TEXT_SMS']);
        elseif($send_sms['error'] == 'no_url')
            return array('fatal_error' => $global_languages['NO_URL']);
        elseif($send_sms['error'] == 'no_vcard')
            return array('fatal_error' => $global_languages['NO_VCARD']);
        elseif($send_sms['error'] == 'not_registr_originator')
            return array('fatal_error' => $global_languages['NOT_REGISTR_ORIGINATOR']);
        elseif($send_sms['error'] == 'not_control_originator')
            return array('fatal_error' => $global_languages['NOT_CONTROL_ORIGINATOR']);
        elseif($send_sms['error'] == 'already_send')
            return array('error' => $global_languages['SMS_ALREADY_SEND']);
        else
            return FALSE;
    }


    // Добавление отправителя SMS на моддерацию и отправка сообщения менеджеру клиента
    // $originator - отправитель SMS
    // $auth_user - ID пользователя
    // возвращает ID отправителя добавленного на модерацию
    function add_originator($originator, $auth_user, $all_originator = 0)
    {
        global $SQL;
        $query = 'select id_originator, state from sms_originator where originator="' . $originator . '" and id_user="' . $auth_user . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $state = $SQL->result($result, 0, 'state');
            if($state == 'completed')
                return 0;
            else
            {
                $id_originator = $SQL->result($result, 0, 'id_originator');
                return $id_originator;
            }
        }
        else
        {
/*
            $query = 'select id_originator from sms_originator where originator="' . $originator . '" and state="rejected"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
                $state = 'rejected';
            else
                $state = 'order';
*/
            if($all_originator)
                $state = 'completed';
            else
                $state = 'order';

            $query = 'insert into sms_originator set id_user="' . $auth_user . '", originator="' . $originator . '", state="' . $state . '"';
            $SQL->query($query);
            $id_originator = $SQL->insert_id();
            return $id_originator;
        }
    }

    // Отправка писем и SMS с сайта с логотипом
    // $address - E-mail получателя
    // $smtp_username - E-mail отправителя
    // $subject - тема письма
    // $message - HTML сообщение
    // $text_message - текстовое сообщение
    function send_mail_logo($address, $smtp_username, $subject, $message, $text_message)
    {
        $message .= '';

        $text_message .= '';

        $subject = change_charset('UTF-8', 'Windows-1251', $subject);
        $message = change_charset('UTF-8', 'Windows-1251', $message);
        $text_message = change_charset('UTF-8', 'Windows-1251', $text_message);

        $hdrs = array('From' => $smtp_username, 'Subject' => $subject, 'Content-Type' => 'multipart/mixed; charset=windows-1251');
        $crlf = "\r\n";
        $mime = new Mail_mime($crlf);

        $mime->setTXTBody($text_message);
        $mime->addHTMLImage('img/logo.gif', 'image/gif;', '', TRUE, '786032112@07102008-2276');
        $mime->setHTMLBody($message);

        $message = $mime->get();
        $hdrs = $mime->headers($hdrs);
        $mail =& Mail::factory('mail');
        $mail->send($address, $hdrs, $message);
    }

    // преобразует число с текстовый формат
    function num2strRU($number)
    {
       global $global_languages;
        //числа от 1 до 9
        $first['1'] = $global_languages['ONE'];
        $first['2'] = $global_languages['TWO'];
        $first['3'] = $global_languages['THREE'];
        $first['4'] = $global_languages['FOUR'];
        $first['5'] = $global_languages['FIVE'];
        $first['6'] = $global_languages['SIX'];
        $first['7'] = $global_languages['SEVEN'];
        $first['8'] = $global_languages['EIGHT'];
        $first['9'] = $global_languages['NINE'];

        //числа от 10 до 19
        $second['10'] = $global_languages['TEN'];
        $second['11'] = $global_languages['ELEVEN'];
        $second['12'] = $global_languages['TWELVE'];
        $second['13'] = $global_languages['THIRTEEN'];
        $second['14'] = $global_languages['FOURTEEN'];
        $second['15'] = $global_languages['FIFTEEN'];
        $second['16'] = $global_languages['SIXTEEN'];
        $second['17'] = $global_languages['SEVENTEEN'];
        $second['18'] = $global_languages['EIGHTEEN'];
        $second['19'] = $global_languages['NINETEEN'];

        //числа от 20 до 100
        $deci['2'] = $global_languages['TWENTY'];
        $deci['3'] = $global_languages['THIRTY'];
        $deci['4'] = $global_languages['FORTY'];
        $deci['5'] = $global_languages['FIFTY'];
        $deci['6'] = $global_languages['SIXTY'];
        $deci['7'] = $global_languages['SEVENTY'];
        $deci['8'] = $global_languages['EIGHTY'];
        $deci['9'] = $global_languages['NINETY'];

        //числа от 200 до 900
        $hund['1'] = $global_languages['ONE_HUNDRED'];
        $hund['2'] = $global_languages['TWO_HUNDRED'];
        $hund['3'] = $global_languages['THREE_HUNDRED'];
        $hund['4'] = $global_languages['FOUR_HUNDRED'];
        $hund['5'] = $global_languages['FIVE_HUNDRED'];
        $hund['6'] = $global_languages['SIX_HUNDRED'];
        $hund['7'] = $global_languages['SEVEN_HUNDRED'];
        $hund['8'] = $global_languages['EIGHT_HUNDRED_AND'];
        $hund['9'] = $global_languages['NINE_HUNDRED_AND'];

        //числа от 1000 до 100.000
        $thou['1'] = $global_languages['THOUSAND'];
        $thou['2'] = $global_languages['THOUSANDS_OF'];
        $thou['3'] = $global_languages['THOUSAND'];
        if($number<1000000)
        {
            $str = floor($number);
            $p = 1;
            $digs = strlen($str);
            while($p<=$digs)
            {
                switch($p)
                {
                    //шаг первый (единицы)
                    case 1:
                        if($digs>1)
                        {
                            //проверка 10 - 19
                            if(substr($str, -2, 1)!=0)
                            {
                                $chk = substr($str, -2, 1)*10 + substr($str, -1, 1);
                                if($chk >=10 and $chk <=19)
                                    $strResult .= $second[$chk].' рублей';
                                elseif(substr($chk,0,1)>=2) //больше либо равно 20
                                {
                                    $strResult .= $deci[substr($chk,0,1)].' ';
                                    if(substr($str, -1, 1)!=0)
                                    {
                                        $strResult .= $first[substr($str, -1, 1)];
                                        $strResult .= lastDigs(substr($str, -1, 1));
                                    }
                                    else
                                        $strResult .= ' рублей';
                                }
                            }
                            elseif(substr($str,-1,1)!=0)
                            {
                                $strResult .= $first[substr($str, -1, 1)];
                                $strResult .= lastDigs(substr($str, -1, 1));
                            }
                            else
                                $strResult .= ' рублей';
                        }
                        else
                        {
                            $strResult .= $first[substr($str, -1, 1)];
                            $strResult .= lastDigs(substr($str, -1, 1));
                        }
                        $p = 3;
                        break;
                    //сотни
                    case 3:
                        if(substr($str, -3, 1)!=0)
                            $strResult = $hund[substr($str, -3, 1)].' '.$strResult;
                        $p = 4;
                        break;
                    //тысячи
                    case 4:
                        if($digs==4)
                            $strResult = thouSwitch(substr($str,-4,1), $thou, $first).$strResult;
                        elseif($digs==5 or $digs==6)
                        {
                            if(substr($str,-5,1)==1)
                            {
                                $thnd = 10 + substr($str,-4,1);
                                $strResult = $second[$thnd].' тысяч '.$strResult;
                            }
                            elseif(substr($str,-5,1)!=0)
                            {
                                if(substr($str,-4,1)==0)
                                    $strResult = $deci[substr($str,-5,1)].' тысяч '.$strResult;
                                else
                                     $strResult = $deci[substr($str,-5,1)].' '.thouSwitch(substr($str,-4,1), $thou, $first).$strResult;
                            }
                        }
                        $p = 6;
                        break;
                    //сотни тысяч
                    case 6:
                        if(substr($str,-5,1)==0 and substr($str,-4,1)!=0) $strResult = thouSwitch(substr($str,-4,1), $thou, $first).$strResult;
                        if(substr($str,-5,1)==0 and substr($str,-4,1)==0) $strResult = ' тысяч '.$strResult;
                        $strResult = $hund[substr($str, -6, 1)].' '.$strResult;
                        $p = 7;
                        break;
                }
            }

            $copeck = substr(number_format($number, 2, ',', ''), -2, 2);
            $ord_copeck = $copeck%10;
            $strResult .= ' ' . $copeck;
            if($ord_copeck == 1 && $copeck != 11)
                $strResult .= ' копейка';
            elseif($ord_copeck >= 5 || $ord_copeck == 0)
                $strResult .= ' копеек';
            else
                $strResult .= ' копейки';

        }
        else
            $strResult = $global_languages['MORE_MLLION_RUB'];

        return $strResult;
    }

    //подставляем окончание фразы, в зависимости от последнего числа
    function lastDigs ($dig)
    {
       global $global_languages;
        switch($dig)
        {
            case 1:
                $out = $global_languages['PHP_RUB'];
                break;
            case 2:
            case 3:
            case 4:
                $out = $global_languages['PHP_RUBLE'];
                break;
            case 5:
            case 6:
            case 7:
            case 8:
            case 9:
                $out = $global_languages['PHP_RUBLES'];
                break;
        }
        return $out;
    }

    //вывод количества тысяч до 9000
    function thouSwitch ($dig, $thou, $first)
    {

       global $global_languages;
        if($dig==1)
            $out = $global_languages['ONE_A'].$thou['1'].' ';
        else
        {
            switch($dig)
            {
                case 2: $thnd = $global_languages['TWO_E']; $curdig = '2'; break;
                case 3: $thnd = $global_languages['THREE']; $curdig = '2'; break;
                case 4: $thnd = $global_languages['FOUR']; $curdig = '2'; break;
                case 5: $thnd = $global_languages['FIVE']; $curdig = '2'; break;
                case 6: $thnd = $global_languages['SIX']; $curdig = '2'; break;
                case 7: $thnd = $global_languages['SEVEN']; $curdig = '2'; break;
                case 8: $thnd = $global_languages['EIGHT']; $curdig = '2'; break;
                case 9: $thnd = $global_languages['NINE']; $curdig = '2'; break;
                $thnd = $first[$dig];
                $curdig = '3';
                break;
            }
            $out = $thnd.' '.$thou[$curdig].' ';
        }
        return $out;
    }

    // $date - дата в формате SQL
    // $month - количество месяцев, которое должно добавить к дате
    // $day - количество дней, которое должно добавить к дате
    // возвращает дату в формате SQL с учетом добавленных дней и месяцев
    function date_package($date, $month = 0, $day = 0)
    {
        $date = mktime(0, 0, 0, substr($date, 5, 2)+$month, 1, substr($date, 0, 4)) + $day*86400;
        $date = date('Y-m-d', $date);
        return $date;
    }

    // $date - дата в формате SQL
    // $month - количество месяцев, которое должно добавить к дате
    // $day - количество дней, которое должно добавить к дате
    // возвращает дату в формате SQL с учетом добавленных дней и месяцев
    function date_add_m_d($date, $month = 0, $day = 0)
    {
        $date = mktime(0, 0, 0, substr($date, 5, 2)+$month, substr($date, 8, 2), substr($date, 0, 4)) + $day*86400;
        $date = date('Y-m-d', $date);
        return $date;
    }

    // создание пакета с оплатой по факту
    // $id_user - ID пользователя
    // $type - месяц, на который создается пакет. 'current' - текущий, остальные значения - следующий месяц
    function create_package_down_payment($id_user, $type)
    {
        global $SQL;
        if($type == 'current')
        {
            $date = date_package(date('Y-m-d'), 0, 0);
            $date_stop = date_package(date('Y-m-d'), +1, -1);
        }
        else
        {
            $date = date_package(date('Y-m-d'), +1, 0);
            $date_stop = date_package(date('Y-m-d'), +2, -1);
        }
        if(check_package_date($id_user, $date) === FALSE)
        {
            $min_price_sms = get_price_sms($auth_user);
            $query = 'insert into sms_package_user set id_user="' . $id_user . '", limit_price="0", pay="0", credit="1", price_sms="' . $min_price_sms['ru']['price_sms'] . '", date_start="' . $date . '", date_stop="' . $date_stop . '"';
            $SQL->query($query);
        }
    }

    // проверка запущены ли копии скрипта, если запущены скрипт завершается
    // $script - название файла скрипта с доп параметрами если заданы
    function check_script($script)
    {
        $cmd = 'ps ajxww | egrep "' . CHECK_SCRIPT . '/' . $script . '"';
        unset($res);
        exec($cmd, $res);
        $count_res = count($res);
        if($count_res > 1)
            exit();
    }

    function delete_empty_value($array)
    {
        $res = array();
        if(is_array($array))
        {
            foreach($array as $value)
            {
                if($value)
                    $res[] = $value;
            }
        }
        return $res;
    }

    // Возвращает название месяца в именительном падеже
    // $num - порядковый номер месяца
    function month_site($num)
    {
       global $global_languages;
        switch((int) $num) {
                    case 1: $period = $global_languages['B_JANUARY']; break;
                    case 2: $period = $global_languages['B_FEBRUARY']; break;
                    case 3: $period = $global_languages['B_MARCH']; break;
                    case 4: $period = $global_languages['B_APRIL']; break;
                    case 5: $period = $global_languages['B_MAY']; break;
                    case 6: $period = $global_languages['B_JUNE']; break;
                    case 7: $period = $global_languages['B_JULY']; break;
                    case 8: $period = $global_languages['B_AUGUST']; break;
                    case 9: $period = $global_languages['B_SEPTEMBER']; break;
                    case 10: $period = $global_languages['B_OCTOBER']; break;
                    case 11: $period = $global_languages['B_NOVEMBER']; break;
                    case 12: $period = $global_languages['B_DECEMBER']; break;
                    default: $period = '?';
                }
                return $period;
    }

    /**
     * Проверка имени отправителя
     * @param string $originator имя отправителя
     * @return boolean
     */
    function check_originator($originator)
    {
        $originator = stripslashes($originator);
        $alfanum = preg_match('/\D/', $originator);
        $len_orig = strlen($originator);
        if($len_orig>0 && (($alfanum && $len_orig<=11) || (!$alfanum && $len_orig<=15)))
            if(preg_match('/^[0-9a-zA-Z\'?><,\.\-_=\+\/"!@#$%^&*() ]+$/', $originator))
                return TRUE;
        return FALSE;
    }

    function delete_xls_user($auth_user)
    {
        if(file_exists('base/' . $auth_user . '.xls'))
            unlink('base/' . $auth_user . '.xls');
    }
    
    function select_param_phone($param_base, $sheet, $selected, $id_base=0)
    {
       global $global_languages, $sms_id_partner;       
        $type_select = 'little';
        ${'selected_'.($selected+1)} = ' selected';

        $i = -1;

        $class_name = 'sheets_' . $sheet . '_' . $selected;
        $return = '<script>var ' . $class_name . ' = new select(\'' . $class_name . '\', \'sheets[' . $sheet . '][]\', \''.$type_select.'\');';
        $return .= $class_name . '.value[' . (++$i) . '] = ["", "'.$global_languages['NOT_DOWNLOAD'].'", ""];';
        if($id_base == -1){
            ${'selected_'.$sheet} = ' selected';
            $return .= $class_name . '.value[' . (++$i) . '] = ["inn", "'.$global_languages['SETTINGS_INN'].'", "' . ${'selected_1'} . '"];';
            $return .= $class_name . '.value[' . (++$i) . '] = ["legal_entity", "'.$global_languages['LEGAL_ENTITY'].'", "' . ${'selected_2'} . '"];';
            $return .= $class_name . '.value[' . (++$i) . '] = ["originator", "'.$global_languages['NAME_ORIGINATOR'].'", "' . ${'selected_3'} . '"];';
            $return .= $class_name . '.value[' . (++$i) . '] = ["comments", "'.$global_languages['COMENT'].'", "' . ${'selected_4'} . '"];';
        } else
            $return .= $class_name . '.value[' . (++$i) . '] = ["phone", "'.$global_languages['TELEPHONE_NAMBER'].'", "' . $selected_1 . '"];';

        if(is_array($param_base))
        {
            foreach($param_base as $param => $name)
            {
                $return .= $class_name . '.value[' . (++$i) . '] = ["' . $param . '", "'.$name.'", "' . ${'selected_'.$i} . '"];';
            }
        }
        $return .= $class_name . '.scroll_num_row = 10;';
        $return .= $class_name . '.Write();</script>';
        return $return;
    }

    function get_sheet_xls($id_base, $tpl, $data)
    {
        $max_rows = 6;
        $sheets = array();
        $num_sheets = count($data->sheets);
        $num_sheets_r = 0;
        $sheet_data = FALSE;
        for($sheet=0;$sheet<$num_sheets;$sheet++)
        {
            if($data->sheets[$sheet]['numCols'])
            {
                $sheets[$num_sheets_r]['name'] = $data->boundsheets[$sheet]['name'];
                $sheets[$num_sheets_r]['numCols'] = $data->sheets[$sheet]['numCols'];
                if($data->sheets[$sheet]["numRows"] > $max_rows)
                    $data->sheets[$sheet]["numRows"] = $max_rows;

                for($row=1; $row<=$data->sheets[$sheet]["numRows"]; $row++)
                {
                    for($col=1; $col<=$data->sheets[$sheet]["numCols"]; $col++)
                    {
                        $sheets[$num_sheets_r]['cells'][$row-1][$col-1] = check_xls_value(nl2br(htmlentities($data->sheets[$sheet]["cells"][$row][$col], ENT_COMPAT, 'UTF-8')));
                        if($sheet_data === FALSE)
                            $sheet_data = $num_sheets_r;
                    }
                }
                $num_sheets_r++;
            }
        }
        get_sheet($id_base, $tpl, $sheets);
        return array('num_sheets' => $num_sheets_r, 'sheet_data' => $sheet_data);
    }
/*
        foreach($xlsx->sheets as $sheet => $rows)
        {
            list($num_cols, $num_rows) = $xlsx->dimension($sheet);
            $rows = $xlsx->rows($sheet);
*/
    function get_sheet_xlsx($id_base, $tpl, $file)
    {
        $xlsx = new SimpleXLSX($file);
        $max_rows = 6;
        $sheets = array();
        $num_sheets = $xlsx->sheetsCount();
        $num_sheets_r = 0;
        $sheet_data = FALSE;
        for($sheet=0;$sheet<$num_sheets;$sheet++)
        {
            list($num_cols, $num_rows) = $xlsx->dimension($sheet+1);
            if($num_cols && $num_rows)
            {
                $sheets[$num_sheets_r]['name'] = 'Лист ' . ($sheet+1);
                $sheets[$num_sheets_r]['numCols'] = $num_cols;
                if($num_rows > $max_rows)
                    $num_rows = $max_rows;

                $rows = $xlsx->rows($sheet+1);
                for($j=0; $j<$num_rows; $j++)
                {
                    for($i=0; $i<$num_cols; $i++)
                    {
                        $sheets[$num_sheets_r]['cells'][$j][$i] = check_xls_value(nl2br(htmlentities($rows[$j][$i], ENT_COMPAT, 'UTF-8')));
                        if(strlen($sheets[$num_sheets_r]['cells'][$j][$i])=='5' && is_numeric($sheets[$num_sheets_r]['cells'][$j][$i]))
                            $sheets[$num_sheets_r]['cells'][$j][$i] .= ' / '.date("d.m.Y", ($sheets[$num_sheets_r]['cells'][$j][$i]-25569)*86400);
                        if($sheet_data === FALSE)
                            $sheet_data = $num_sheets_r;
                    }
                }
                $num_sheets_r++;
            }
        }
        get_sheet($id_base, $tpl, $sheets);
        return array('num_sheets' => $num_sheets_r, 'sheet_data' => $sheet_data);
    }

    function get_array_phone($string)
    {
        $len_string = mb_strlen($string);
        $new_line = 0;
        $phone = '';
        for($i=0; $i<$len_string; $i++)
        {
            $letter = get_phone(mb_substr($string, $i, 1));
            if($letter !== FALSE)
            {
                $phone .= $letter;
                $new_line = 1;
            }
            elseif($new_line)
            {
                $array[] = $phone;
                $phone = '';
                $new_line = 0;
            }
        }
        if($new_line)
            $array[] = $phone;
        return $array;
    }

    function convert_txt($tmp_name, $new_file)
    {
        $fp = fopen($tmp_name, 'r');
        $new_fp = fopen($new_file, 'w');
        while(($string = fgets($fp, 3072000)) !== FALSE)
        {
            $string = iconv('Windows-1251', 'UTF-8', $string);
            $len_string = mb_strlen($string);
            $new_line = 0;
            for($i=0; $i<$len_string; $i++)
            {
                $letter = get_phone(mb_substr($string, $i, 1));
                if($letter !== FALSE)
                {
                    fwrite($new_fp, $letter);
                    $new_line = 1;
                }
                elseif($new_line)
                {
                    fwrite($new_fp, "\r\n");
                    $new_line = 0;
                }
            }
            if($new_line)
                fwrite($new_fp, "\r\n");
        }
        fclose($fp);
        fclose($new_fp);
    }

    function get_sheet_csv($id_base, $tpl, $file)
    {
       global $global_languages;
        $max_rows = 6;
        $sheets = array();
        $sheets[0]['name'] = $global_languages['SHEET'].' 1';

        $row = 0;
        $num_cols = 0;
        $fp = fopen($file, 'r');
        while(($string = fgets($fp, 3072000)) !== FALSE && $row<$max_rows)
        {
            $string = iconv('Windows-1251', 'UTF-8', $string);
            $data = explode(';', $string);
            $num = count($data);
            if($num > $num_cols)
                $num_cols = $num;
            for($c=0; $c < $num; $c++)
            {
                if(substr($data[$c], 0, 1) == '"')
                    $data[$c] = substr($data[$c], 1);
                if(substr($data[$c], -1) == '"')
                    $data[$c] = substr($data[$c], 0, -1);
                $data[$c] = str_replace('""', '"', $data[$c]);
                $sheets[0]['cells'][$row][$c] = check_xls_value(nl2br(htmlentities($data[$c], ENT_COMPAT, 'UTF-8')));
            }
            $row++;
        }
        $sheets[0]['numCols'] = $num_cols;
        fclose ($fp);
        get_sheet($id_base, $tpl, $sheets);
        return array('num_sheets' => 1, 'sheet_data' => 0);
    }

    function get_sheet($id_base, $tpl, $data)
    {
        $param_base = get_param_id_base($id_base);
        $num_sheets = count($data);
        for($sheet=0;$sheet<$num_sheets;$sheet++)
        {
            $tpl->get('SHEET', array('NUMBER_SHEET'), array($sheet), 1);
            $tpl->get('HEADER', array('HEADER', 'NUMBER_SHEET'), array($data[$sheet]['name'], $sheet), 1);

            $max_rows = count($data[$sheet]['cells']);
            if($max_rows && $data[$sheet]['numCols'])
            {
                for($i=0; $i<$data[$sheet]['numCols']; $i++)
                {
                    $tpl->get('HEAD', array('HEAD'), array(select_param_phone($param_base, $sheet, $i, $id_base)), 1);
                }
                $tpl->delete_section('HEAD');

                for($row=0; $row<$max_rows; $row++)
                {
                    $tpl->get('ROW_DATA', array('ROW'), array($row+1), 1);
                    for($col=0; $col<$data[$sheet]["numCols"]; $col++)
                    {
                        $tpl->get('DATA', array('DATA'), array($data[$sheet]["cells"][$row][$col]), 1);
                    }
                    $tpl->delete_section('DATA');
                }
                $tpl->delete_section('ROW_DATA');
            }
            else
                $tpl->get('HEAD', array('HEAD'), array('Лист не содержит данных. Выберите другой лист.'));
        }
    }

    function add_phone_xls($id_base, $sheets, $data, $id_background)
    {
        global $SQL;
        $sheet_count = 0;
        for($sheet=0;$sheet<count($data->sheets);$sheet++)
        {
            if(isset($sheets[$sheet_count]))
            {
                $rest_array[$sheet_count] = $data->sheets[$sheet]["numRows"];
            }
            if($data->sheets[$sheet]['numCols'])
                $sheet_count++;
        }
        $base_info=array();
        $phones_insert = array();
        $query = 'select id_base, double_phone from sms_base where id_base="' . $id_base . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result)>0)
        {
            $base_info = $SQL->fetch_assoc($result);
        }
        $count_load = 0;
        $num_load = 0;
        $sheet_count = 0;
        set_phone_code_in_array();
        for($sheet=0;$sheet<count($data->sheets);$sheet++)
        {
            if(isset($sheets[$sheet_count]))
            {
            $count_rows = $data->sheets[$sheet]["numRows"];
            for($row=1;$row<=$data->sheets[$sheet]["numRows"]&&($row<=$max_rows||$max_rows==0);$row++)
            {
                $phones = array();
                for($col=1;$col<=$data->sheets[$sheet]["numCols"]&&($col<=$max_cols||$max_cols==0);$col++)
                {
                    if($data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"] >=1 && $data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"] >=1)
                    {
                        $this_cell_colspan = ' colspan=' . $data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];
                        $this_cell_rowspan = ' rowspan=' . $data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"];
                        for($i=1;$i<$data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];$i++)
                        {
                            $data->sheets[$sheet]["cellsInfo"][$row][$col+$i]["dontprint"]=1;
                        }
                        for($i=1;$i<$data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"];$i++)
                        {
                            for($j=0;$j<$data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];$j++)
                            {
                                $data->sheets[$sheet]["cellsInfo"][$row+$i][$col+$j]["dontprint"]=1;
                            }
                        }
                    }
                    elseif($data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"] >=1)
                    {
                        $this_cell_colspan = ' colspan=' . $data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];
                        $this_cell_rowspan = '';
                        for($i=1;$i<$data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];$i++)
                        {
                            $data->sheets[$sheet]["cellsInfo"][$row][$col+$i]["dontprint"]=1;
                        }
                    }
                    elseif($data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"] >=1)
                    {
                        $this_cell_colspan = '';
                        $this_cell_rowspan = ' rowspan=' . $data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"];
                        for($i=1;$i<$data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"];$i++)
                        {
                            $data->sheets[$sheet]["cellsInfo"][$row+$i][$col]["dontprint"]=1;
                        }
                    }
                    else
                    {
                        $this_cell_colspan = '';
                        $this_cell_rowspan = '';
                    }

                    if(!($data->sheets[$sheet]["cellsInfo"][$row][$col]["dontprint"]))
                    {
                        $phones[$col-1] = nl2br(htmlentities($data->sheets[$sheet]["cells"][$row][$col], ENT_COMPAT, 'UTF-8'));
                    }
                }

                if(is_array($sheets[$sheet_count]))
                {
                    $i = 0;
                    $phone = '';
                    $region = '';
                    $operator = '';
                    $name = '';
                    $surname = '';
                    $patronymic = '';
                    $date_birth = '';
                    $male = '';
                    $addition_1 = '';
                    $addition_2 = '';
                    $addition_3 = '';
                    $addition_4 = '';
                    $addition_5 = '';
                    foreach($sheets[$sheet_count] as $key => $val)
                    {
                        if($val)
                            ${$val} = check_xls_value($phones[$key]);
                        $i++;
                    }
                    if($count_rows>500){
                        if($base_info["double_phone"]=='1'){
                            $phones_insert_in = $name.'-'.$surname.'-'.$patronymic.'-'.$addition_1.'-'.$addition_2.'-'.$addition_3.'-'.$addition_4.'-'.$addition_5;
                            if($phones_insert[get_replace_phone($phone)]!=$phones_insert_in)
                            {
                                $phones_insert[get_replace_phone($phone)]=$phones_insert_in;
                                $count_load++;
                                $num_load += add_phone($id_base, $phone, $region, $operator, $name, $surname, $patronymic, $date_birth, $male, $addition_1, $addition_2, $addition_3, $addition_4, $addition_5);
                            }                
                        } else {
                            if($phones_insert[get_replace_phone($phone)]=='')
                            {
                                $phones_insert[get_replace_phone($phone)]='1';
                                $count_load++;
                                $num_load += add_phone($id_base, $phone, $region, $operator, $name, $surname, $patronymic, $date_birth, $male, $addition_1, $addition_2, $addition_3, $addition_4, $addition_5);
                            }
                        }
                    } else {
                        if($base_info["double_phone"]=='1'){
                            $query = 'select id_phone, phone from sms_phone where id_base="' . $id_base . '" and phone="'.get_replace_phone($phone).'" and name="'.$name.'" and surname="'.$surname.'" and patronymic="'.$patronymic.'" and addition_1="'.$addition_1.'" and addition_2="'.$addition_2.'" and addition_3="'.$addition_3.'" and addition_4="'.$addition_4.'" and addition_5="'.$addition_5.'"';
                            $result = $SQL->query($query);
                            if($SQL->num_rows($result)==0)
                            {
                                $count_load++;
                                $num_load += add_phone($id_base, $phone, $region, $operator, $name, $surname, $patronymic, $date_birth, $male, $addition_1, $addition_2, $addition_3, $addition_4, $addition_5);
                            }                
                        } else {
                            $query = 'select id_phone, phone from sms_phone where id_base="' . $id_base . '" and phone="'.get_replace_phone($phone).'"';
                            $result = $SQL->query($query);
                            if($SQL->num_rows($result)==0)
                            {
                                $count_load++;
                                $num_load += add_phone($id_base, $phone, $region, $operator, $name, $surname, $patronymic, $date_birth, $male, $addition_1, $addition_2, $addition_3, $addition_4, $addition_5);
                            }
                        }
                    }
                }
                $rest_array[$sheet_count]--;
                if($count_load%847 == 0)
                    set_num_sms($id_background, 1, $rest_array, $num_load);
            }
            }
            if($data->sheets[$sheet]['numCols'])
                $sheet_count++;
        }
        $num_load += add_phone();
        set_num_sms($id_background, 1, $rest_array, $num_load);
        if($count_load)
            insert_update('base', $id_base);
        return $num_load;
    }

    function add_phone_xlsx($id_base, $sheets, $file, $id_background)
    {
        global $SQL;
        set_phone_code_in_array();
        if(file_exists($file))
        {
        $xlsx = new SimpleXLSX($file);

        $sheet_count = 0;
        $num_sheets = $xlsx->sheetsCount();
        for($sheet=0;$sheet<$num_sheets;$sheet++)
        {
            list($num_cols, $num_rows) = $xlsx->dimension($sheet+1);
            if(isset($sheets[$sheet_count]))
                $rest_array[$sheet_count] = $num_rows;
            if($num_cols && $num_rows)
                $sheet_count++;
        }
        $base_info=array();
        $phones_insert = array();
        $query = 'select id_base, double_phone from sms_base where id_base="' . $id_base . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result)>0)
        {
            $base_info = $SQL->fetch_assoc($result);
        }
        $count_load = 0;
        $num_load = 0;
        $sheet_count = 0;
        $num_sheets = $xlsx->sheetsCount();
        for($sheet=0;$sheet<$num_sheets;$sheet++)
        {
            list($num_cols, $num_rows) = $xlsx->dimension($sheet+1);
            if(isset($sheets[$sheet_count]))
            {
            $rows = $xlsx->rows($sheet+1);
            for($j=0; $j<$num_rows; $j++)
            {
                $phones = array();
                for($i=0; $i<$num_cols; $i++)
                {
                    $phones[$i] = nl2br(htmlentities($rows[$j][$i], ENT_COMPAT, 'UTF-8'));
                }
                if(is_array($sheets[$sheet_count]))
                {
                    $i = 0;
                    $phone = '';
                    $region = '';
                    $operator = '';
                    $name = '';
                    $surname = '';
                    $patronymic = '';
                    $date_birth = '';
                    $male = '';
                    $addition_1 = '';
                    $addition_2 = '';
                    $addition_3 = '';
                    $addition_4 = '';
                    $addition_5 = '';
                    foreach($sheets[$sheet_count] as $key => $val)
                    {
                        if($val && $phones[$key])
                        {
                            if($val == 'date_birth' && strlen($phones[$key])=='5')
                            {
                                $phones[$key] = date("d.m.Y", ($phones[$key]-25569)*86400);
                            }
                            ${$val} = check_xls_value($phones[$key]);
                        }
                        $i++;
                    }
                    if($num_rows>500){
                        if($base_info["double_phone"]=='1'){
                            $phones_insert_in = $name.'-'.$surname.'-'.$patronymic.'-'.$addition_1.'-'.$addition_2.'-'.$addition_3.'-'.$addition_4.'-'.$addition_5;
                            if($phones_insert[get_replace_phone($phone)]!=$phones_insert_in)
                            {
                                $phones_insert[get_replace_phone($phone)]=$phones_insert_in;
                                $count_load++;
                                $num_load += add_phone($id_base, $phone, $region, $operator, $name, $surname, $patronymic, $date_birth, $male, $addition_1, $addition_2, $addition_3, $addition_4, $addition_5);
                            }                
                        } else {
                            if($phones_insert[get_replace_phone($phone)]=='')
                            {
                                $phones_insert[get_replace_phone($phone)]='1';
                                $count_load++;
                                $num_load += add_phone($id_base, $phone, $region, $operator, $name, $surname, $patronymic, $date_birth, $male, $addition_1, $addition_2, $addition_3, $addition_4, $addition_5);
                            }
                        }
                    } else {
                        if($base_info["double_phone"]=='1'){
                            $query = 'select id_phone, phone from sms_phone where id_base="' . $id_base . '" and phone="'.get_replace_phone($phone).'" and name="'.$name.'" and surname="'.$surname.'" and patronymic="'.$patronymic.'" and addition_1="'.$addition_1.'" and addition_2="'.$addition_2.'" and addition_3="'.$addition_3.'" and addition_4="'.$addition_4.'" and addition_5="'.$addition_5.'"';
                            $result = $SQL->query($query);
                            if($SQL->num_rows($result)==0)
                            {
                                $count_load++;
                                $num_load += add_phone($id_base, $phone, $region, $operator, $name, $surname, $patronymic, $date_birth, $male, $addition_1, $addition_2, $addition_3, $addition_4, $addition_5);
                            }                
                        } else {
                            $query = 'select id_phone, phone from sms_phone where id_base="' . $id_base . '" and phone="'.get_replace_phone($phone).'"';
                            $result = $SQL->query($query);
                            if($SQL->num_rows($result)==0)
                            {
                                $count_load++;
                                $num_load += add_phone($id_base, $phone, $region, $operator, $name, $surname, $patronymic, $date_birth, $male, $addition_1, $addition_2, $addition_3, $addition_4, $addition_5);
                            }
                        }
                    }
                }
                $rest_array[$sheet_count]--;
                if($count_load%847 == 0)
                    set_num_sms($id_background, 1, $rest_array, $num_load);
            }
            }
            if($num_cols && $num_rows)
                $sheet_count++;
        }
        }
        $num_load += add_phone();
        set_num_sms($id_background, 1, $rest_array, $num_load);
        if($count_load)
            insert_update('base', $id_base);
        return $num_load;
    }

    function add_phone_csv($id_base, $sheets, $file, $id_background)
    {
        global $SQL;
        set_phone_code_in_array();
        if(file_exists($file))
        {
        $base_info=array();
        $phones_insert = array();
        $query = 'select id_base, double_phone from sms_base where id_base="' . $id_base . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result)>0)
        {
            $base_info = $SQL->fetch_assoc($result);
        }
        $count_load = 0;
        $num_load = 0;
        $fp = fopen($file, 'r');

        $rest = 0;
        while(($string = fgets($fp, 3072000)) !== FALSE && isset($sheets[0]))
        {
            $rest++;
        }
        rewind($fp);

        while(($string = fgets($fp, 3072000)) !== FALSE && isset($sheets[0]))
        {
            $phones = array();
            $string = iconv('Windows-1251', 'UTF-8', $string);
            $data = explode(';', $string);
            $num = count($data);
            for($c=0; $c < $num; $c++)
            {
                if(substr($data[$c], 0, 1) == '"')
                    $data[$c] = substr($data[$c], 1);
                if(substr($data[$c], -1) == '"')
                    $data[$c] = substr($data[$c], 0, -1);
                $data[$c] = str_replace('""', '"', $data[$c]);

                $phones[$c] = htmlentities($data[$c], ENT_COMPAT, 'UTF-8');
            }

            if(is_array($sheets[0]))
            {
                $i = 0;
                $phone = '';
                $region = '';
                $operator = '';
                $name = '';
                $surname = '';
                $patronymic = '';
                $date_birth = '';
                $male = '';
                $addition_1 = '';
                $addition_2 = '';
                $addition_3 = '';
                $addition_4 = '';
                $addition_5 = '';
                foreach($sheets[0] as $key => $val)
                {
                    if($sheets[0][$key])
                        ${$sheets[0][$key]} = check_xls_value($phones[$key]);
                    $i++;
                }
                if($rest>500){
                    if($base_info["double_phone"]=='1'){
                        $phones_insert_in = $name.'-'.$surname.'-'.$patronymic.'-'.$addition_1.'-'.$addition_2.'-'.$addition_3.'-'.$addition_4.'-'.$addition_5;
                        if($phones_insert[get_replace_phone($phone)]!=$phones_insert_in)
                        {
                            $phones_insert[get_replace_phone($phone)]=$phones_insert_in;
                            $count_load++;
                            $num_load += add_phone($id_base, $phone, $region, $operator, $name, $surname, $patronymic, $date_birth, $male, $addition_1, $addition_2, $addition_3, $addition_4, $addition_5);
                        }                
                    } else {
                        if($phones_insert[get_replace_phone($phone)]=='')
                        {
                            $phones_insert[get_replace_phone($phone)]='1';
                            $count_load++;
                            $num_load += add_phone($id_base, $phone, $region, $operator, $name, $surname, $patronymic, $date_birth, $male, $addition_1, $addition_2, $addition_3, $addition_4, $addition_5);
                        }
                    }
                } else {
                    if($base_info["double_phone"]=='1'){
                        $query = 'select id_phone, phone from sms_phone where id_base="' . $id_base . '" and phone="'.get_replace_phone($phone).'" and name="'.$name.'" and surname="'.$surname.'" and patronymic="'.$patronymic.'" and addition_1="'.$addition_1.'" and addition_2="'.$addition_2.'" and addition_3="'.$addition_3.'" and addition_4="'.$addition_4.'" and addition_5="'.$addition_5.'"';
                        $result = $SQL->query($query);
                        if($SQL->num_rows($result)==0)
                        {
                            $count_load++;
                            $num_load += add_phone($id_base, $phone, $region, $operator, $name, $surname, $patronymic, $date_birth, $male, $addition_1, $addition_2, $addition_3, $addition_4, $addition_5);
                        }                
                    } else {
                        $query = 'select id_phone, phone from sms_phone where id_base="' . $id_base . '" and phone="'.get_replace_phone($phone).'"';
                        $result = $SQL->query($query);
                        if($SQL->num_rows($result)==0)
                        {
                            $count_load++;
                            $num_load += add_phone($id_base, $phone, $region, $operator, $name, $surname, $patronymic, $date_birth, $male, $addition_1, $addition_2, $addition_3, $addition_4, $addition_5);
                        }
                    }
                }
            }
            $rest--;
            if($count_load%847 == 0)
                set_progress_background($id_background, $rest, $num_load);
        }
        fclose ($fp);
        }
        $num_load += add_phone();
        set_progress_background($id_background, $rest, $num_load);
        if($count_load)
            insert_update('base', $id_base);
        return $num_load;
    }

    function add_stop_phone_xls($id_user, $data, $type = '')
    {
        $count_load = 0;
        for($sheet=0;$sheet<count($data->sheets);$sheet++)
        {
            for($row=1;$row<=$data->sheets[$sheet]["numRows"]&&($row<=$max_rows||$max_rows==0);$row++)
            {
                $phones = array();
                for($col=1;$col<=$data->sheets[$sheet]["numCols"]&&($col<=$max_cols||$max_cols==0);$col++)
                {
                    if($data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"] >=1 && $data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"] >=1)
                    {
                        $this_cell_colspan = ' colspan=' . $data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];
                        $this_cell_rowspan = ' rowspan=' . $data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"];
                        for($i=1;$i<$data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];$i++)
                        {
                            $data->sheets[$sheet]["cellsInfo"][$row][$col+$i]["dontprint"]=1;
                        }
                        for($i=1;$i<$data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"];$i++)
                        {
                            for($j=0;$j<$data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];$j++)
                            {
                                $data->sheets[$sheet]["cellsInfo"][$row+$i][$col+$j]["dontprint"]=1;
                            }
                        }
                    }
                    elseif($data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"] >=1)
                    {
                        $this_cell_colspan = ' colspan=' . $data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];
                        $this_cell_rowspan = '';
                        for($i=1;$i<$data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];$i++)
                        {
                            $data->sheets[$sheet]["cellsInfo"][$row][$col+$i]["dontprint"]=1;
                        }
                    }
                    elseif($data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"] >=1)
                    {
                        $this_cell_colspan = '';
                        $this_cell_rowspan = ' rowspan=' . $data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"];
                        for($i=1;$i<$data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"];$i++)
                        {
                            $data->sheets[$sheet]["cellsInfo"][$row+$i][$col]["dontprint"]=1;
                        }
                    }
                    else
                    {
                        $this_cell_colspan = '';
                        $this_cell_rowspan = '';
                    }

                    if(!($data->sheets[$sheet]["cellsInfo"][$row][$col]["dontprint"]))
                    {
                        $count_load += add_phone_stop($id_user, check_xls_value($data->sheets[$sheet]["cells"][$row][$col]), $type);
                    }
                }
            }
        }
        if($count_load)
            insert_update('stop', $id_user);
        return $count_load;
    }

    function add_stop_phone_xlsx($id_user, $file, $type = '')
    {
        $xlsx = new SimpleXLSX($file);
        $count_load = 0;
        $sheet_count = 0;

        $num_sheets = $xlsx->sheetsCount();
        for($sheet=0;$sheet<$num_sheets;$sheet++)
        {
            list($num_cols, $num_rows) = $xlsx->dimension($sheet+1);
            $rows = $xlsx->rows($sheet+1);
            for($j=0; $j<$num_rows; $j++)
            {
                $phones = array();
                for($i=0; $i<$num_cols; $i++)
                {
                    $count_load += add_phone_stop($id_user, check_xls_value($rows[$j][$i]), $type);
                }
            }
            if($num_cols && $num_rows)
                $sheet_count++;
        }
        if($count_load)
            insert_update('stop', $id_user);
        return $count_load;
    }

    function add_stop_phone_csv($id_user, $file, $type = '')
    {
        $count_load = 0;
        $fp = fopen($file, 'r');
        while(($string = fgets($fp, 3072000)) !== FALSE)
        {
            $phones = array();
            $string = iconv('Windows-1251', 'UTF-8', $string);
            $data = explode(';', $string);
            $num = count($data);
            for($c=0; $c < $num; $c++)
            {
                if(substr($data[$c], 0, 1) == '"')
                    $data[$c] = substr($data[$c], 1);
                if(substr($data[$c], -1) == '"')
                    $data[$c] = substr($data[$c], 0, -1);
                $data[$c] = str_replace('""', '"', $data[$c]);

                $count_load += add_phone_stop($id_user, $data[$c], $type);
            }
        }
        fclose ($fp);

        if($count_load)
            insert_update('stop', $id_user);
        return $count_load;
    }

    function add_phone_stop($id_user, $phone, $type)
    {
        global $SQL;
        $phone = get_replace_phone($phone);
        if($phone)
        {
            if($type == 'global')
                $query = 'insert into sms_global_stop set phone="' . $phone . '"';
            else
                $query = 'insert into sms_stop set id_user="' . $id_user . '", phone="' . $phone . '"';
            $SQL->query($query);
            if($SQL->affected_rows()>0)
                return 1;
            else
                return 0;
        }
        else
            return 0;
    }

    function check_xls_value($value)
    {
        $values = explode('E+', $value);
        if(count($values) == 2 && is_numeric($values[0]) && is_numeric($values[1]))
            $value = (int)($values[0]*pow(10,$values[1]));
        $values = explode('/', $value);
        if(count($values) == 3 && is_numeric($values[0]) && is_numeric($values[1]) && is_numeric($values[2]))
            $value = date_add_day_xls($values[2].'-'.$values[1].'-'.$values[0], -1);
        return $value;
    }

    function get_region($operator, $region, $phone)
    {
        global $SQL, $global_region, $global_operator;
        if(!$region)
        {
            $query = 'select region, time_zone from sms_def_code where code_from<=' . $phone . ' and code_to>=' . $phone;
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
            {
                $region = $SQL->result($result, 0, 'region');
                $time_zone = $SQL->result($result, 0, 'time_zone');
            }
            else
                $region = '';
        }
        
        $MCC_MNC = get_phone_code_from_array($phone);
        if(!$operator)
        {
            if($MCC_MNC === FALSE)
                $operator = '';
            elseif(($param_operator = get_param_phone_code($MCC_MNC['MCC'], $MCC_MNC['MNC'])) === FALSE)
                $operator = '';
            else
                $operator = $param_operator['title'];
        }

        if(isset($global_region[$region]))
            $id_region = $global_region[$region];
        else
        {
            if($region)
            {
                $query = 'select id_region from sms_phone_region where region="' . addslashes($region) . '"';
                $result = $SQL->query($query);
                if($SQL->num_rows($result))
                    $id_region = $SQL->result($result, 0, 'id_region');
                else
                {
                    $query = 'insert into sms_phone_region set region="' . addslashes($region) . '"';
                    $SQL->query($query);
                    $id_region = $SQL->insert_id();
                }
                $global_region[$region] = $id_region;
            }
            else
                $id_region = 0;
        }

        if(isset($global_operator[$operator]))
            $id_operator = $global_operator[$operator];
        else
        {
            if($operator)
            {
                $query = 'select id_operator from sms_phone_operator where operator="' . addslashes($operator) . '"';
                $result = $SQL->query($query);
                if($SQL->num_rows($result))
                    $id_operator = $SQL->result($result, 0, 'id_operator');
                else
                {
                    $query = 'insert into sms_phone_operator set operator="' . addslashes($operator) . '"';
                    $SQL->query($query);
                    $id_operator = $SQL->insert_id();
                }
                $global_operator[$operator] = $id_operator;
            }
            else
                $id_operator = 0;
        }

        return array('id_region' => $id_region, 'id_operator' => $id_operator, 'region' => $region, 'operator' => $operator, 'time_zone' => $time_zone, 'MCC' => $MCC_MNC['MCC'], 'MNC' => $MCC_MNC['MNC']);
    }

    function add_phone($id_base = FALSE, $phone = '', $region = '', $operator = '', $name = '', $surname = '', $patronymic = '', $date_birth = '', $male = '', $addition_1 = '', $addition_2 = '', $addition_3 = '', $addition_4 = '', $addition_5 = '')
    {
        global $SQL, $global_add_phone, $global_add_phone_counter;

        if($global_add_phone_counter > 1000 || ($id_base === FALSE && $global_add_phone))
        {
            $global_add_phone_counter = 0;
            $global_add_phone = substr($global_add_phone, 0, -1);
            $SQL->query($global_add_phone);
            $num_rows = $SQL->affected_rows();
        }

        $phone = get_replace_phone($phone);
        if($phone)
        {
            $def = get_region($operator, $region, $phone);
            $name = addslashes($name);
            $surname = addslashes($surname);
            $patronymic = addslashes($patronymic);
            $addition_1 = addslashes($addition_1);
            $addition_2 = addslashes($addition_2);
            $addition_3 = addslashes($addition_3);
            $addition_4 = addslashes($addition_4);
            $addition_5 = addslashes($addition_5);
            if($date_birth)
            {
                $date_birth = str_replace('/', '.', $date_birth);
                $date_birth = preg_replace("/([0-9]{4})\-([0-9]{2})\-([0-9]{2})/", "$3.$2.$1", $date_birth);
                $date_birth = date_site_admin_sql($date_birth);
            }
            else
                $date_birth = '0000-00-00';
            $male = mb_strtolower($male);// Переводим в нижний регистр
            if(!$male)
                $male = 'NULL';
            elseif((strrpos($male, 'м') !== FALSE)||(strrpos($male, 'м') === 0))
                $male = '"m"';
            else
                $male = '"f"';
            if(!$global_add_phone_counter)
                $global_add_phone = 'insert ignore into sms_phone (id_base, phone, id_region, id_operator, name, surname, patronymic, date_birth, male, addition_1, addition_2, addition_3, addition_4, addition_5, MCC, MNC) VALUES';
            $global_add_phone_counter++;
            $global_add_phone .= '("' . $id_base . '", "' . $phone . '", "' . $def['id_region'] . '", "' . $def['id_operator'] . '", "' . $name . '", "' . $surname . '", "' . $patronymic . '", "' . $date_birth . '", ' . $male . ', "' . $addition_1 . '", "' . $addition_2 . '", "' . $addition_3 . '", "' . $addition_4 . '", "' . $addition_5 . '", "' . $def['MCC'] . '", "' . $def['MNC'] . '"),';
        }
        return $num_rows;
    }

    function get_phone_code($id_package, $table)
    {
        global $SQL;
        $return = array();
        $query = 'select sp.price_sms, spc.title from sms_price_' . $table . ' as sp, sms_phone_code as spc where sp.id_package_' . $table . '="' . $id_package . '" and sp.phone_code=spc.phone_code group by spc.title';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($j=0; $j<$num; $j++)
        {
            $price_sms = $SQL->result($result, $j, 'sp.price_sms');
            $title = $SQL->result($result, $j, 'spc.title');
            $return[$title] = $price;
        }
        return $return;
    }

    // формирует набор тегов option по запросу к БД
    // $query - запрос должен возвращать набор данных состоящий из name - отображаемого значения и id - передаваемого значения через форму
    // $id_select - массив значений, которые нужно отметить [id] = 1
    // возвращает HTML код сожержащий теги option
    function get_option_phone_code($id_select = FALSE)
    {
        global $SQL;
        $query = 'select phone_code, title from sms_phone_code where order by title asc';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($i=0; $i < $num; $i++)
        {
            $title = $SQL->result($result, $i, 'title');
            $phone_code = $SQL->result($result, $i, 'phone_code');
            if(isset($return[$title]))
                $return[$title] = ',' . $phone_code;
            else
                $return[$title] = $phone_code;
        }

        $option = '';
        for($i=0; $i < $num; $i++)
        {
            $name = $SQL->result($result, $i, 'name');
            $id = $SQL->result($result, $i, 'id');
            if(is_array($id_select) && $id_select[$id])
                $selected = ' selected';
            else
                $selected = '';
            $id = str_replace('"', '&quot;',$id);
            $option .= '<option value="' . $id . '"' . $selected . '>' . $name . '</option>';
        }
        return $option;
    }

    // формирует массив значений которые нужно отметить
    // $select - массив значений которые нужно отметить [fild]
    // возвращает массив [fild] = ' selected'
    function get_selected($select)
    {
        $selected = array();
        if(is_array($select))
        {
            foreach($select as $fild)
            {
                $selected[$fild] = ' selected';
            }
        }
        return $selected;
    }

    // $type - название поля
    // $select - массив значений которые могут содержаться в SQL результате
    // возвращает SQL условие для запроса
    function get_where($type, $select)
    {
        $where = '';
        if(is_array($select))
        {
            foreach($select as $fild)
            {
                $where .= $type . '="' . $fild . '" or ';
            }
        }
        if($where)
            $where = '(' . $where . '0) and ';
        return $where;
    }

    function get_where_MCC($table, $MCC,$MNC)
    {
        $where = '';
        if(is_array($MCC))
        {
            foreach($MCC as $key=>$fild)
            {
                $where .= '(' . $table . '.MCC ="' . $fild . '"';
                if($MNC[$key])
                     $where .= ' and ' . $table . '.MNC ="' . $MNC[$key] . '"';
                $where .=') or ';
            }
        }

        if($where)
            $where = '(' . $where . '0) and ';
        return $where;
    }

    // $type - название поля
    // $select - массив значений которые могут содержаться в SQL результате
    // возвращает SQL условие для запроса
    function get_where_before($type, $select)
    {
        $where = '';
        if(is_array($select))
        {
            $where = ' and (0';
            foreach($select as $fild)
            {
                $where .= ' or ' . $type . '="' . $fild . '"';
            }
            $where .= ')';
        }
        return $where;
    }

    // формирует набор тегов option по запросу к БД
    // $query - запрос должен возвращать набор данных состоящий из name - отображаемого значения и id - передаваемого значения через форму
    // $id_select - массив значений, которые нужно отметить [id] = 1
    // возвращает HTML код сожержащий теги option
    function get_multi_select($query, $id_select = FALSE, $no_select = FALSE)
    {
        global $SQL;
        if($no_select)
            $option .= '<option value="">' . $no_select . '</option>';
        else
            $option = '';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($i=0; $i < $num; $i++)
        {
            $name = $SQL->result($result, $i, 'name');
            $id = $SQL->result($result, $i, 'id');
            if(is_array($id_select) && isset($id_select[$id]))
                $selected = ' selected';
            else
                $selected = '';
            $id = str_replace('"', '&quot;',$id);
            $option .= '<option value="' . $id . '"' . $selected . '>' . $name . '</option>';
        }
        return $option;
    }

    // формирует набор тегов option по запросу к БД
    // $query - запрос должен возвращать набор данных состоящий из name - отображаемого значения и id - передаваемого значения через форму
    // $id_select - массив значений, которые нужно отметить [id] = 1
    // возвращает HTML код сожержащий теги option
    function get_multi_select_tpl($tpl, $section, $query, $id_select = FALSE, $default_options=FALSE)
    {
        global $SQL;
        $j=0;
        if(is_array($default_options))
        {
            foreach($default_options as $value)
            {
                if(is_array($id_select) && isset($id_select[$value['id']]))
                    $selected = ' selected';
                else
                    $selected = '';
                $tpl->get($section, array('COUNT', 'ID', 'NAME', 'SELECTED'), array($j, $value['id'], $value['name'], $selected), 1);
                $j++;
            }
        }

        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($i=0; $i < $num; $i++)
        {
            $name = $SQL->result($result, $i, 'name');
            $id = $SQL->result($result, $i, 'id');
            if(is_array($id_select) && isset($id_select[$id]))
                $selected = ' selected';
            else
                $selected = '';
            $id = str_replace('"', '\"', $id);
            $name = str_replace('"', '\"', $name);
            $tpl->get($section, array('COUNT', 'ID', 'NAME', 'SELECTED'), array($i+$j, $id, $name, $selected), 1);
        }
    }

    // формирует набор тегов option из массива
    // $param - массив значаний name - отображаемого значения и id - передаваемого значения через форму
    // $id_select - массив значений, которые нужно отметить [id] = 1
    // возвращает HTML код сожержащий теги option
    function get_multi_select_from_array($param, $id_select = FALSE)
    {
        $option = '';
        $num = count($param);
        for($i=0; $i < $num; $i++)
        {
            $name = $param[$i]['name'];
            $id = $param[$i]['id'];
            if(is_array($id_select) && isset($id_select[$id]))
                $selected = ' selected';
            else
                $selected = '';
            $id = str_replace('"', '&quot;',$id);
            $option .= '<option value="' . $id . '"' . $selected . '>' . $name . '</option>';
        }
        return $option;
    }
    
    /**
     * формирует набор тегов option из массива
     * @param array $param массив значаний name - отображаемого значения и id - передаваемого значения через форму
     * @param array $id_select массив значений, которые нужно отметить [id] = 1
     * @param type $no_select
     * @return string возвращает HTML код сожержащий теги option
     */
    function get_multi_select_from_array_noselect($param, $id_select = FALSE, $no_select = FALSE)
    {
        if($no_select)
            $option .= '<option value="">' . $no_select . '</option>';
        else
            $option = '';
        $num = count($param);
        for($i=0; $i < $num; $i++)
        {
            $name = $param[$i]['name'];
            $id = $param[$i]['id'];
            if(is_array($id_select) && isset($id_select[$id]))
                $selected = ' selected';
            else
                $selected = '';
            $id = str_replace('"', '&quot;',$id);
            $option .= '<option value="' . $id . '"' . $selected . '>' . $name . '</option>';
        }
        return $option;
    }

    function get_multi_select_from_array_tpl($tpl, $section, $param, $id_select = FALSE)
    {
        $num = count($param);
        for($i=0; $i < $num; $i++)
        {
            $name = $param[$i]['name'];
            $id = $param[$i]['id'];
            if(is_array($id_select) && isset($id_select[$id]))
                $selected = ' selected';
            else
                $selected = '';
            $id = str_replace('"', '\"', $id);
            $name = str_replace('"', '\"', $name);
            $tpl->get($section, array('COUNT', 'ID', 'NAME', 'SELECTED'), array($i, $id, $name, $selected), 1);
        }
    }

    function get_set_from_array($param)
    {
        $option = '';
        if(is_array($param))
        {
            foreach($param as $p)
            {
                if($p != '')
                    $option .= '"' . $p . '",';
            }
        }
        if($option)
            $option = substr($option, 0, -1);
        return $option;
    }


    function get_set_from_key_array($param)
    {
        $option = '';
        if(is_array($param))
        {
            foreach($param as $key => $val)
            {
                if($val)
                    $option .= '"' . $key . '", ';
            }
        }
        $option = substr($option, 0, -2);
        return $option;
    }

    function mkGraph($qry, $width, $height, $title)
    {
        global $SQL;
        $gr = new gGroupedBarChart;
            $gr->width = $width;
            $gr->height = $height;
            $res = $SQL->query($qry);
            unset($data);
            $sum = 0;
            $i = 0;
            while ($row = $SQL->fetch_row($res)) {
                $sum += $data[$i]['value'] = $row[0];
                $data[$i]['label'] = $row[1];
                $i++;
            }
            if(is_array($data))
            {
                foreach($data as $value)
                {
                    if($sum)
                        $persent = round($value['value']*100/$sum, 1);
                    else
                        $persent = 0;
                    $gr->addDataSet(array(round($persent, 0)));
                    $dsCountLab[] = $value['label'] . ' - ' . $persent . '% (' .$value['value'] . ')';
                }
            }

            $gr->setHorizontal(true);
            $gr->setTitle($title);
            $gr->valueLabels = $dsCountLab;
            $gr->dataColors = array('2263c5', '37a402', 'a543a1', '0081a9', 'ff0312', 'ffa224', 'c907b5');
            $uri = $gr->getURL();
//            return $uri;
            return substr($uri, 0, strpos($gr->getURL(), '&chxr')).substr($uri, strpos($uri, '&chbh'));
    }

    function mkGraphArray($values, $width, $height, $title)
    {
        $gr = new gGroupedBarChart;
        $gr->width = $width;
        $gr->height = $height;
        unset($data);
        $sum = 0;
        $i = 0;
        if(is_array($values))
        {
            foreach($values as $value)
            {
                $sum += $data[$i]['value'] = $value['data'];
                $data[$i]['label'] = $value['legend'];
                $i++;
            }
        }

            if(is_array($data))
            {
                foreach($data as $value)
                {
                    if($sum)
                        $persent = round($value['value']*100/$sum, 1);
                    else
                        $persent = 0;
                    $gr->addDataSet(array(round($persent, 0)));
                    $dsCountLab[] = $value['label'] . ' - ' . $persent . '% (' .$value['value'] . ')';
                }
            }

            $gr->setHorizontal(true);
            $gr->setTitle($title);
            $gr->valueLabels = $dsCountLab;
            $gr->dataColors = array('2263c5', '37a402', 'a543a1', '0081a9', 'ff0312', 'ffa224', 'c907b5');
            $uri = $gr->getURL();
//            return $uri;
            return substr($uri, 0, strpos($gr->getURL(), '&chxr')).substr($uri, strpos($uri, '&chbh'));
    }

    function send_mail_admin_help($id_help_title, $id_help, $login, $text)
    {
       global $SQL, $global_languages;
        $message = $global_languages['QUESTION_FROM_USER'].': ' . $login . '

' . $text . '

http://' . HTTP_SERVER . '/lkadm/section.php?id_page=56&id_module_page=145&section=new&id_help=' . $id_help;

        $choice = 'select e_mail, phone from help_title where id_help_title="' . $id_help_title . '"';
        $result = $SQL->query($choice);
        if($SQL->num_rows($result))
        {
            $e_mail = $SQL->result($result, 0, 'e_mail');
            $phone = $SQL->result($result, 0, 'phone');
            send_sms_site($phone, $login . ' ' . $global_languages['ASK_IN_HELP']);
            send_mail($e_mail, HTTP_SERVER . ' '.$global_languages['QUESTION_FROM_USER'], $message);
        }
    }
    
    function send_sms_site($phone, $text)
    {
        global $param_site;
        $phone_array = get_array_phone($phone);
        if(is_array($phone_array))
        {
            foreach($phone_array as $phone_)
            {
                $phone_ = get_phone($phone_);
                if($phone_)
                    send_sms(ID_SMS_SITE, 1, 0, 0, $phone_, $param_site['SENDER_SERVICE_SMS'], 'sms', $text);
            }
        }
    }

    function send_sms_mail_admin($message, $title, $mail_message, $type = 'privilege')
    {
        send_mail_admin($title, $mail_message, $type);
        send_sms_admin($title, $message, $type);
    }

    function send_mail_admin($title, $mail_message, $type = 'privilege')
    {
        global $SQL, $param_site;

        $title = HTTP_SERVER . ' ' . $title;
        $choice = 'select * from admin_users';
        $result = $SQL->query($choice);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++)
        {
            $e_mail = $SQL->result($result, $i, 'e_mail');
            $send = $SQL->result($result, $i, $type);
            if($send)
                send_mail($e_mail, $title, $mail_message, 'text/plain; charset=windows-1251', 1);
        }
    }

    function send_mail_partner($id_partner, $title, $mail_message)
    {
        global $SQL, $param_site;
        if($id_partner)
        {
            $choice = 'select e_mail, domain from sms_partners where id_partner="' . $id_partner . '"';
            $result = $SQL->query($choice);
            $num = $SQL->num_rows($result);
            if($num)
            {
                $e_mail = $SQL->result($result, 0, 'e_mail');
                $domain = $SQL->result($result, 0, 'domain');
                send_mail($e_mail, $domain . ' ' . $title, $mail_message);
            }
        }
        else
            send_mail_admin($title, $mail_message);
    }

    function send_sms_admin($message, $type = 'privilege')
    {
        global $SQL, $param_site;

        $choice = 'select * from admin_users';
        $result = $SQL->query($choice);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++)
        {
            $phone = $SQL->result($result, $i, 'phone');
            $send = $SQL->result($result, $i, $type);
            if($send)
                send_sms(ID_SMS_SITE, 1, 0, 0, $phone, $param_site['SENDER_SERVICE_SMS'], 'sms', $message);
        }
    }

    function send_sms_mail_manager($id_manager, $message, $title, $mail_message, $type = FALSE)
    {
        send_mail_manager($id_manager, $title, $mail_message, 'send_e_mail_' . $type);
        send_sms_manager($id_manager, $message, 'send_sms_' . $type);
    }

    function send_mail_manager($id_manager, $title, $mail_message, $type = FALSE)
    {
        global $SQL, $param_site;

        $choice = 'select * from sms_manager where id_manager="' . $id_manager . '"';
        $result = $SQL->query($choice);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++)
        {
            $e_mail = $SQL->result($result, $i, 'e_mail');
            $id_partner = $SQL->result($result, $i, 'id_partner');
            if($id_partner)
            {
                $choice2 = 'select domain from sms_partners where id_partner="' . $id_partner . '"';
                $result2 = $SQL->query($choice2);
                if($SQL->num_rows($result2))
                    $domain = $SQL->result($result2, 0, 'domain');
                else
                    $domain = HTTP_SERVER;
            }
            else
                $domain = HTTP_SERVER;
            $title = $domain . ' ' . $title;
            $e_mail_notification = $SQL->result($result, $i, 'e_mail_notification');
            if($e_mail_notification)
                $e_mail = $e_mail_notification;
            if($type)
                $send = $SQL->result($result, $i, $type);
            if($type === FALSE || $send)
                send_mail($e_mail, $title, $mail_message);
        }
    }

    function send_sms_manager($id_manager, $message, $type = FALSE)
    {
        global $SQL, $param_site;

        $choice = 'select * from sms_manager where id_manager="' . $id_manager . '"';
        $result = $SQL->query($choice);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++)
        {
            $phone = $SQL->result($result, $i, 'phone_notification');
            if($type)
                $send = $SQL->result($result, $i, $type);
            if($type === FALSE || $send)
                send_sms_site($phone, $message);
        }
    }

    function send_sms_partner($id_partner, $message)
    {
        global $SQL, $param_site;
        if($id_partner)
        {
            $choice = 'select phone from sms_partners where id_partner="' . $id_partner . '"';
            $result = $SQL->query($choice);
            $num = $SQL->num_rows($result);
            if($num)
            {
                $phone = $SQL->result($result, 0, 'phone');
                send_sms(ID_SMS_SITE, 1, 0, 0, $phone, $param_site['SENDER_SERVICE_SMS'], 'sms', $message);
            }
        }
        else
            send_sms_admin($message);
    }

    function send_partner_mail($id_user, $id_module_page, $id_partner, $param = '')
    {
        global $SQL;
        $choice = "select title, text from sms_title_text_partner where id_module_page='" . $id_module_page . "' and id_partner='" . $id_partner . "'";
        $result = $SQL->query($choice);
        if($SQL->num_rows($result))
        {
            $mail_message = $SQL->result($result, 0, 'text');
            $mail_title = $SQL->result($result, 0, 'title');
            $tpl_mail = new tpl($mail_message, 'string');
            if($mail_message!='')
            {
                if($id_partner)
                {
                    $choice = 'select domain from sms_partners where id_partner="' . $id_partner . '"';
                    $result = $SQL->query($choice);
                    if($SQL->num_rows($result))
                        $domain = $SQL->result($result, 0, 'domain');
                    else
                        $domain = HTTP_SERVER;
                }
                else
                    $domain = HTTP_SERVER;

                $query = 'select * from sms_send_users where id_user="' . $id_user . '"';
                $result = $SQL->query($query);
                $num = $SQL->num_rows($result);
                if($num)
                {
                    $login = $SQL->result($result, 0, 'login');
                    $name = magic_quotes_gpc($SQL->result($result, 0, 'name'));
                    $company_name = $SQL->result($result, 0, 'company_name');
                    $phone = $SQL->result($result, 0, 'phone');
                    $e_mail = $SQL->result($result, 0, 'e_mail');
                    $forget = 'http://' . $domain . '/reg/122/restoration/' . $id_user . '/' . $SQL->result($result, 0, 'forget') . '.html';

                    $p = array('LOGIN', 'NAME', 'COMPANY_NAME', 'PHONE', 'E_MAIL', 'FORGET');
                    $v = array($login, $name, $company_name, $phone, $e_mail, $forget);
                    if(is_array($param))
                    {
                        foreach($param as $key => $val)
                        {
                            $p[] = $key;
                            $v[] = $val;
                        }
                    }
                    $mail_message = $tpl_mail->get_value($mail_message, $p, $v, count($p));
                    send_mail($e_mail, $mail_title, $mail_message,'text/html; charset=windows-1251');
                }
                return TRUE;
            } else
                return FALSE;
        }
        else
            return FALSE;
    }

    function send_mail($e_mail, $title, $mail_message, $content_type = 'text/plain; charset=windows-1251', $support=0)
    {
        global $param_site;
        if($param_site['SMTP_HOST'] && $param_site['SMTP_PORT'])
        {
            if($param_site['SMTP_USER']=='info@incoredevelopment.com' and $support != 1){
                $content_type = 'text/html; charset=windows-1251';
                $mail_message = send_mail_incoredevelopment($mail_message, $title);
            }
            $title = change_charset('UTF-8', 'Windows-1251', $title);
            $mail_message = change_charset('UTF-8', 'Windows-1251', $mail_message);
            $hdrs = array('From' => $param_site['SENDER_SERVICE_E_MAIL'], 'To' => $e_mail, 'Subject' => $title, 'Content-Type' => $content_type);
            $mime = new Mail_mime("\r\n");
            $mime->_build_params = array(
                'html_charset'  => 'Windows-1251',
                'text_charset'  => 'Windows-1251',
                'head_charset'  => 'Windows-1251',
            );
            $mime->setTXTBody($mail_message);
            $hdrs = $mime->headers($hdrs);
            $message = $mime->get();
            $mail =& Mail::factory('smtp', array('host' => $param_site['SMTP_HOST'], 'port' => $param_site['SMTP_PORT'], 'auth' => 1, 'username' => $param_site['SMTP_USER'], 'password' =>  $param_site['SMTP_PASSWORD']));
//            $mail->debug = TRUE;
            $mail->send($e_mail, $hdrs, $message);
        }
    }

    function get_part_sms($text_sms, $part)
    {
        $text_sms = str_replace("\r\n", "\n", $text_sms);
        $text_sms = stripslashes($text_sms);
        $len_text_sms = mb_strlen($text_sms);
        $encoding = get_char_message($text_sms);
        if($encoding == 0x8)
        {
            if($len_text_sms > 70)
                $len_sms = 67;
            else
                $len_sms = 70;
        }
        else
        {
            $len_text_sms1 = strlen(convert_encoding($text_sms, 0x0, TRUE)); // GSM
            $len_text_sms2 = strlen(convert_encoding($text_sms, 0x3, TRUE)); // Латиница
            if($len_text_sms1 > $len_text_sms2)
                $len_text_sms = $len_text_sms1;
            else
                $len_text_sms = $len_text_sms2;

            if($len_text_sms > 160)
                $len_sms= 153;
            else
                $len_sms = 160;
        }
        $part = mb_substr($text_sms, ($part-1)*$len_sms, $len_sms);
        return $part;
    }

    // Замена переменных номера телефона в тексте SMS
    function get_text($text_sms, $phone, $name='', $surname='', $patronymic='', $date_birth='', $male='', $addition_1='', $addition_2='', $addition_3='', $addition_4='', $addition_5='')
    {
       global $global_languages;
        $text_sms = str_replace('#number#', '+'.$phone, $text_sms);
        $text_sms = str_replace('#last name#', $surname, $text_sms);
        $text_sms = str_replace('#name#', $name, $text_sms);
        $text_sms = str_replace('#middle name#', $patronymic, $text_sms);
        $date_birth = date_sql_site_admin($date_birth);
        $text_sms = str_replace('#birth#', $date_birth, $text_sms);
        if($male == 'm')
        {
            $male_ = $global_languages['MAN_PHP'];
            $end_1 = $global_languages['IJ'];
        }
        elseif($male == 'f')
        {
            $male_ = $global_languages['WOMAN_PHP'];
            $end_1 = $global_languages['AA'];
        }
        else
        {
            $male_ = '';
            $end_1 = $global_languages['IJ'];
        }
        $text_sms = str_replace('#sex#', $male_, $text_sms);
        $text_sms = str_replace('#end_word#', $end_1, $text_sms);
        $text_sms = str_replace('#note1#', $addition_1, $text_sms);
        $text_sms = str_replace('#note2#', $addition_2, $text_sms);
        $text_sms = str_replace('#note3#', $addition_3, $text_sms);
        $text_sms = str_replace('#note4#', $addition_4, $text_sms);
        $text_sms = str_replace('#note5#', $addition_5, $text_sms);
        return $text_sms;
    }

    function get_text_sms($text_sms)
    {
        $text_sms = str_replace("\r\n", "\n", $text_sms);
        $text_sms = str_replace("«", '\"', $text_sms);
        $text_sms = str_replace("»", '\"', $text_sms);
        $text_sms = str_replace("“", '\"', $text_sms);
        $text_sms = str_replace("”", '\"', $text_sms);
        $text_sms = str_replace("`", '\"', $text_sms);
        return $text_sms;
    }

    function htmlspecialcharsdecode($str)
    {
        $str = str_replace('&amp;', '&', $str);
        $str = str_replace('&quot;', '"', $str);
        $str = str_replace('&#039;', "'", $str);
        $str = str_replace('&lt;', '<', $str);
        $str = str_replace('&gt;', '>', $str);
        return $str;
    }

    function send_xml_error($str)
    {
        $doc = '<?xml version="1.0" encoding="utf-8"?>
<response>
    <error>' . $str . '</error>
</response>';
        return $doc;
    }

    function send_from_xml($type, $xml)
    {
       global $global_languages;
        $log = new Log();
        $log->make_dir(LOG_XML . 'error');
        $log->log_file_name = LOG_XML . 'error/' . $type;
        $log->get_data($global_languages['RECEIVE_PACKAGE']."\n");
        $log->get_data("ip ". preg_replace('/[^0-9.]/', '', $_SERVER['REMOTE_ADDR'])."\n\n\n");
        $in = trim($xml);
        $log->get_data($in);
        if($in)
        {
            if($type == 'originator'|| $type=='time')
                $function = 'get_balance_from_xml';
            else
                $function = 'get_' . $type . '_from_xml';
            $xml = $function($in);  // Выполнил эту функцию
            if($xml == FALSE)
            {
                $log->set_data();
                $resp = send_xml_error('Неправильный формат XML документа');
            }
            else
            {
                $log->get_data('Массив значаний');
                $log->get_data($xml);
                if(!empty($xml['login']))
                {
                    $log->make_dir(LOG_XML . preg_replace('/[^0-9A-Za-z_]/', '', $xml['login']));
                    $log->log_file_name = LOG_XML . preg_replace('/[^0-9A-Za-z_]/', '', $xml['login']).'/state';
                    $log->set_data();
                }

                $function = 'send_' . $type . '_from_array';
                $answer = $function($xml);
                $function = 'resp_' . $type . '_from_array';
                $resp = $function($answer); // ответ пользователю
                $log->get_data('Ответ');
                $log->get_data($resp);
                $log->get_data("\r\n\r\n");
            }
        }
        else
            $resp = send_xml_error($global_languages['POST_DATA_NOT_AVAILABLE']);
        $log->set_data();

        return $resp;
    }

    function get_sms_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;

        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        foreach($xml['message'] as $i=>$mess)
        {
            $type = $mess['a']['type'];
            if($type != 'flashsms' && $type != 'wappush' && $type != 'vcard')
                $type = 'sms';
            $result['message'][$i]['type'] = $type;
            $result['message'][$i]['name_delivery'] = $mess['a']['name_delivery'];

            if($type == 'sms' || $type == 'flashsms' || $type == 'wappush')
                $result['message'][$i]['text'] = str_replace('\n', "\n", $mess['v']['text'][0]['v']);

            if($type == 'wappush' || $type == 'vcard')
                $result['message'][$i]['url'] = $mess['v']['url'][0]['v'];

            if($type == 'vcard')
            {
                $result['message'][$i]['name'] = $mess['v']['name'][0]['v'];
                $result['message'][$i]['phone_cell'] = $mess['v']['phone'][0]['a']['cell'];
                $result['message'][$i]['phone_work'] = $mess['v']['phone'][0]['a']['work'];
                $result['message'][$i]['phone_fax'] = $mess['v']['phone'][0]['a']['fax'];
                $result['message'][$i]['e_mail'] = $mess['v']['email'][0]['v'];
                $result['message'][$i]['position'] = $mess['v']['position'][0]['v'];
                $result['message'][$i]['organization'] = $mess['v']['organization'][0]['v'];
                $result['message'][$i]['address_post_office_box'] = $mess['v']['address'][0]['a']['post_office_box'];
                $result['message'][$i]['address_street'] = $mess['v']['address'][0]['a']['street'];
                $result['message'][$i]['address_city'] = $mess['v']['address'][0]['a']['city'];
                $result['message'][$i]['address_region'] = $mess['v']['address'][0]['a']['region'];
                $result['message'][$i]['address_postal_code'] = $mess['v']['address'][0]['a']['postal_code'];
                $result['message'][$i]['address_country'] = $mess['v']['address'][0]['a']['country'];
                $result['message'][$i]['additional'] = $mess['v']['additional'][0]['v'];
            }
//            elseif($type == 'vcalendar')
//                $url = $mess['url'][0];

            $result['message'][$i]['sender'] = $mess['v']['sender'][0]['v'];

            if(is_array($mess['v']['abonent']))
            foreach($mess['v']['abonent'] as $j=>$abonent)
            {
                $number_sms = $abonent['a']['number_sms'];
                $result['message'][$i]['phones'][$number_sms]['phone'] = $abonent['a']['phone'];
                $result['message'][$i]['phones'][$number_sms]['time_send'] = $abonent['a']['time_send'];
                $result['message'][$i]['phones'][$number_sms]['validity_period'] = $abonent['a']['validity_period'];
                $result['message'][$i]['phones'][$number_sms]['client_id_sms'] = $abonent['a']['client_id_sms'];
                if(!$result['message'][$i]['phones'][$number_sms]['client_id_sms'])
                    $result['message'][$i]['phones'][$number_sms]['client_id_sms'] = 0;
            }

            // Для отправки по базе
            if(is_array($mess['v']['base']))
            foreach($mess['v']['base'] as $j=>$abonent)
            {
                if(!is_numeric($abonent['a']['id_base']))
                    continue;
                $number_sms = $abonent['a']['number_sms'];
                $result['message'][$i]['base'][$number_sms]['id_base'] = $abonent['a']['id_base'];
                $result['message'][$i]['base'][$number_sms]['time_send'] = $abonent['a']['time_send'];
                $result['message'][$i]['base'][$number_sms]['validity_period'] = $abonent['a']['validity_period'];
                $result['message'][$i]['base'][$number_sms]['client_id_sms'] = $abonent['a']['client_id_sms'];
                if(!$result['message'][$i]['base'][$number_sms]['client_id_sms'])
                    $result['message'][$i]['base'][$number_sms]['client_id_sms'] = 0;
                $result['message'][$i]['base'][$number_sms]['male'] = $abonent['a']['male'];
                $result['message'][$i]['base'][$number_sms]['old_from'] = $abonent['a']['old_from'];
                $result['message'][$i]['base'][$number_sms]['old_to'] = $abonent['a']['old_to'];
            }

        }
        return $result;
    }

    /**
     * Отправка sms из массива
     * @global type $SQL
     * @global type $global_languages
     * @param array $xml
     * @return boolean|string
     */
    function send_sms_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user, block, all_originator, url_get_state, id_partner, control_text, send_from, send_to, local_time from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result)) {
            //отбиваем заблокированных пользователей
            $block = $SQL->result($result, 0, 'block');
            if($block) {
                $answer['error'] = $global_languages['YOUR_ACCOUNT_BLOCKED'];
                $answer['error_code'] = 'block';
            } else {
                $id_user = $SQL->result($result, 0, 'id_user');
                //проверяем пользователя по email (для отправки по SMTP)
                if(check_user_e_mail($xml['e_mail'], $id_user) === FALSE)
                    return FALSE;
                //проверяем пользователя по ip
                if(empty($xml['e_mail'])){
                    if(check_user_ip_xml($id_user) === FALSE)
                        return array('error'=>$global_languages['INCORRECT_IP_USER'], 'error_code'=>'block');
                }
                $id_partner = $SQL->result($result, 0, 'id_partner');
                $all_originator = $SQL->result($result, 0, 'all_originator');
                $send_from = $SQL->result($result, 0, 'send_from');
                $send_to = $SQL->result($result, 0, 'send_to');
                $control_text = $SQL->result($result, 0, 'control_text');
                $url_get_state = $SQL->result($result, 0, 'url_get_state');
                $local_time = $SQL->result($result, 0, 'local_time');
                if($url_get_state)
                    $send_xml = 1;
                else
                    $send_xml = 0;

                $respone = '';
                $date = date('Y-m-d');
                $year = date('Y');
                $month = date('m');
                $day = date('d');
                set_param_send_sms();
                $query_sms_priority = 'select `sms_priority` from `sms_send_users` where `id_user`="' . $id_user . '"';
                $sms_priority = $SQL->query($query_sms_priority);
                if($SQL->num_rows($sms_priority)){
                    $priority = $SQL->result($sms_priority, 0, 'sms_priority');
                }
                foreach($xml['message'] as $message)
                {
                    if(!$message['name_delivery'])
                        $message['name_delivery'] = 'Шлюз';
                    if(is_array($message['phones']))
                    foreach($message['phones'] as $num_message => $phone)
                    {
                        if($phone['time_send'])
                            $time = $phone['time_send'];
                        else
                            $time = $date . ' 00:00';

                        $time = get_allow_time($time, $send_from, $send_to);
                        $date_send = date_add_minute($time, -$local_time);
                        $date_send = explode(' ', $date_send);
                        $send_sms = send_sms($id_user, $all_originator, $control_text, $id_partner, addslashes($phone['phone']), addslashes($message['sender']), addslashes($message['type']), addslashes($message['text']), addslashes($message['url']), addslashes($message['name']), addslashes($message['phone_cell']), addslashes($message['phone_work']), addslashes($message['phone_fax']), addslashes($message['e_mail']), addslashes($message['position']), addslashes($message['organization']), addslashes($message['address_post_office_box']), addslashes($message['address_street']), addslashes($message['address_city']), addslashes($message['address_region']), addslashes($message['address_postal_code']), addslashes($message['address_country']), addslashes($message['additional']), $date_send[0], $date_send[1], 'Шлюз', 'all', addslashes($phone['client_id_sms']), 0, $send_xml, addslashes($phone['validity_period']), 1, $priority, FALSE);
                        if(is_array($error = get_error($send_sms)))
                        {
                            $answer[$num_message] = $error;
                        }
                        else
                            $answer[$num_message] = array('id_turn'=>$send_sms['id_turn'], 'id_sms' => $send_sms['id_state'], 'parts' => $send_sms['count_sms']);
                    }
                    if(is_array($message['base']))
                    foreach($message['base'] as $num_message => $base)
                    {
                        if($base['time_send'])
                            $time = $base['time_send'];
                        else
                            $time = $date . ' 00:00';

                        $time = get_allow_time($time, $send_from, $send_to);
                        $date_send = date_add_minute($time, -$local_time);
                        $date_send = explode(' ', $date_send);

                        $query = 'select name_base from sms_base where id_base="' . (int)$base['id_base'] . '" and id_user="' . $id_user . '"';
                        $result = $SQL->query($query);
                        if(!$result) continue;
                        if(!$SQL->num_rows($result)) continue;
                        $where_phone='';
                        if($base['old_from'])
                            $where_phone .= ' and sp.date_birth<="' . ($year-$base['old_from']) . '-' . $month . '-' . $day . '"';
                        if($base['old_to'])
                            $where_phone .= ' and sp.date_birth>="' . ($year-$base['old_to']) . '-' . $month . '-' . $day . '"';
                        if($base['male']==='0')
                            $where_phone .= ' and male is NULL ';
                        elseif($base['male']==='m')
                            $where_phone .= ' and male="m" ';
                        elseif($base['male']==='f')
                            $where_phone .= ' and male="f" ';


                        $query = 'select * from sms_phone where id_base="' . (int)$base['id_base'] . '"' . $where_phone;
                        $result = $SQL->query($query);
                        if(!$result) continue;
                        while($res = $SQL->fetch_assoc($result))
                        {
                            $send_sms = send_sms($id_user, $all_originator, $control_text, $id_partner, addslashes($res['phone']), addslashes($message['sender']), addslashes($message['type']), addslashes($message['text']), addslashes($message['url']), addslashes($message['name']), addslashes($message['phone_cell']), addslashes($message['phone_work']), addslashes($message['phone_fax']), addslashes($message['e_mail']), addslashes($message['position']), addslashes($message['organization']), addslashes($message['address_post_office_box']), addslashes($message['address_street']), addslashes($message['address_city']), addslashes($message['address_region']), addslashes($message['address_postal_code']), addslashes($message['address_country']), addslashes($message['additional']), $date_send[0], $date_send[1], 'Шлюз', 'all', addslashes($phone['client_id_sms']), 0, $send_xml, addslashes($phone['validity_period']), 1, $priority, FALSE);
                            if(is_array($error = get_error($send_sms)))
                            {
                                $answer[$num_message]['base'][] = $error;
                            }
                            else
                                $answer[$num_message]['base'][] = array('id_turn'=>$send_sms['id_turn'], 'id_sms' => $send_sms['id_state'], 'parts' => $send_sms['count_sms']);
                        }
                    }
                }
            }
        } else {
            $answer['error'] = $global_languages['YOUR_ACCOUNT_ERROR'];
            $answer['error_code'] = 'wrong_auth';
        }
        return $answer;
    }

    function resp_sms_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer))
            {
                foreach($answer as $num_message => $message)
                {

                    if(isset($message['fatal_error']))
                        $respone .= send_error($message['fatal_error'], $num_message);

                    elseif(isset($message['error']))
                        $respone .= send_error($message['error'], $num_message);
                    elseif(is_array($message['base']))
                    {
                        $respone .= '<information number_sms="' . $num_message . '">';
                        foreach($message['base'] as $sms)
                        {
                            if(isset($message['fatal_error']))
                                $respone .= '<sms>'.$sms['fatal_error'].'</sms>';
                            elseif(isset($message['error']))
                                $respone .= '<sms>'.$sms['error'].'</sms>';
                            else
                                $respone .= '<sms id_sms="' . $sms['id_sms'] . '" id_turn="' . $sms['id_turn'] . '" parts="' . $sms['parts'] . '">send</sms>';
                        }
                        $respone .= '</information>';
                    }
                    else
                        $respone .= '<information number_sms="' . $num_message . '" id_sms="' . $message['id_sms'] . '" id_turn="' . $message['id_turn'] . '" parts="' . $message['parts'] . '">send</information>
';
                }
            }
            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
' . $respone . '</response>';
        }
        return $resp;
    }

     function get_state_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        foreach($xml['get_state'][0]['v']['id_sms'] as $st)
        {
            $result['state'][]['id_sms'] = $st['v'];
        }
        return $result;
    }

    function send_state_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user, all_originator,local_time from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            $local_time = $SQL->result($result, 0, 'local_time');
            if(check_user_ip_xml($id_user))
            {
                $respone = '';
                foreach($xml['state'] as $state)
                {
                    $query = 'select state, time_change_state from sms_state_final where id_user="' . $id_user . '" and id_state="' . (int)$state['id_sms'] . '"';
                    $result = $SQL->query($query);
                    if($SQL->num_rows($result))
                    {
                        $state_sms = $SQL->result($result, 0, 'state');
                        $time_change_state = date_add_minute($SQL->result($result, 0, 'time_change_state'), +$local_time);
                    }
                    else
                    {
                         $state_sms = $global_languages['MESSAGE_NOT_BEEN_TAKEEN'];
                         $time_change_state = '';
                    }
                    if($state_sms == 'partly_deliver')
                        $state_sms = 'send';
                    $return[] = array('id_sms' => (int)$state['id_sms'], 'time_change_state' => $time_change_state, 'state_sms' => $state_sms);
                }
            } else
                $return['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $return['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $return;
    }

    function resp_state_from_array($answer)
    {
        global $global_languages;
        if(isset($answer['error'])){
            $resp = send_xml_error($answer['error']);
            return $resp;
        }
        if(is_array($answer))
        {
            foreach($answer as $state)
            {
                $respone .= '<state id_sms="' . $state['id_sms'] . '" time="' . $state['time_change_state'] . '">' . $state['state_sms'] . '</state>
';
            }
            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
' . $respone . '</response>';
        }
        else
            $resp = send_xml_error($global_languages['INCORRECT_LOGIN_OR_PASSWORD']);
        return $resp;
    }

    function get_balance_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        return $result;
    }


    function send_balance_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user, id_currency from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                $id_currency = $SQL->result($result, 0, 'id_currency');
                $date = date('Y-m-d');
                $rest_sms = get_rest_packages($id_user, $date);
                if(is_array($rest_sms['sms']))
                {
                    foreach($rest_sms['sms'] as $title => $sms)
                    {
                        $resp['sms'][] = array('area' => $title, 'sms' => $sms);
                    }
                }
                $resp['money'] = $rest_sms['money'];
                $currency = get_param_id_currency($id_currency);
                $resp['short_currency'] = $currency['short_title'];
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }
    function send_time_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user, local_time from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                $local_time = $SQL->result($result, 0, 'local_time');
                $date_local = date_add_minute(date('Y-m-d H:i:s'), +$local_time);
                $resp['time'] = $date_local;
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }
    function resp_time_from_array($answer)
    {
        if(isset($answer['error'])){
            $resp = send_xml_error($answer['error']);
            return $resp;
        }
        $resp .= '<time>' . $answer['time'] . '</time>';
        $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';
        return $resp;
    }

    function resp_balance_from_array($answer)
    {
        if(isset($answer['error'])){
            $resp = send_xml_error($answer['error']);
            return $resp;
        }
        if(is_array($answer['sms']))
        {
            foreach($answer['sms'] as $sms)
            {
                $resp .= '<sms area="' . htmlspecialchars($sms['area']) . '">' . $sms['sms'] . '</sms>';
            }
        }
        $resp .= '<money currency="' . $answer['short_currency'] . '">' . $answer['money'] . '</money>';
        $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';
        return $resp;
    }

    function send_originator_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user, all_originator from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                $all_originator = $SQL->result($result, 0, 'all_originator');

                $respone['all_originator'] = $all_originator;
                if(!$all_originator)
                {
                    $query = 'select state, originator from sms_originator where id_user="' . $id_user . '" order by originator asc';
                    $result = $SQL->query($query);
                    $num = $SQL->num_rows($result);
                    for($i=0; $i<$num; $i++)
                    {
                        $respone['list_originator'][$i]['state'] = $SQL->result($result, $i, 'state');
                        $respone['list_originator'][$i]['originator'] = $SQL->result($result, $i, 'originator');
                    }
                }
            } else
                $respone['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $respone['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $respone;
    }

    function send_add_originator_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user, all_originator from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                $all_originator = $SQL->result($result, 0, 'all_originator');

                $query = 'select state, originator from sms_originator where id_user="' . $id_user . '" order by originator asc';
                $result = $SQL->query($query);
                $num = $SQL->num_rows($result);
                $i=0;
                foreach($xml['originator'] as $add)
                {
                    $respone['originator'][$i]['name'] = $add;
                    $respone['originator'][$i]['add'] = $SQL->result($result, $i, 'originator');
                    if(check_originator(addslashes($add)))
                    {
                        add_originator(addslashes($add), $id_user, $all_originator);
                        $respone['originator'][$i]['add'] = $global_languages['SENDER_PLACED_QUEUE'];
                    }
                    else
                        $respone['originator'][$i]['add'] = $global_languages['SENDER_CAN_ONLY_CONSIST'];

                        $i++;
                }
            } else
                $respone['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $respone['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $respone;
    }

    function resp_add_originator_from_array($answer)
    {
        global $global_languages;
        if(isset($answer['error'])){
            $resp = send_xml_error($answer['error']);
            return $resp;
        }
        if($answer)
        {
            $resp .= '<originator>';
            if(is_array($answer['originator']))
            {
                foreach($answer['originator'] as $or)
                {
                    $resp .= '<add name="' . $or['name'] . '">' . htmlspecialchars($or['add']) . '</originator>';
                }
            }
            $resp .= '</originator>';

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';
        }
        else
            $resp = send_xml_error($global_languages['INCORRECT_LOGIN_OR_PASSWORD']);
        return $resp;
    }

    function resp_originator_from_array($answer)
    {
        global $global_languages;
        if(isset($answer['error'])){
            $resp = send_xml_error($answer['error']);
            return $resp;
        }
        if($answer)
        {
            if($answer['all_originator'])
                $resp = '<any_originator>TRUE</any_originator>';
            else
            {
                $resp = '<any_originator>FALSE</any_originator>';
                $resp .= '<list_originator>';
                if(is_array($answer['list_originator']))
                {
                    foreach($answer['list_originator'] as $or)
                    {
                        $resp .= '<originator state="' . $or['state'] . '">' . htmlspecialchars($or['originator']) . '</originator>';
                    }
                }
                $resp .= '</list_originator>';
            }
            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';
        }
        else
            $resp = send_xml_error($global_languages['INCORRECT_LOGIN_OR_PASSWORD']);
        return $resp;
    }

    function get_incoming_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        $result['start'] = $xml['time'][0]['a']['start'];
        $result['end'] = $xml['time'][0]['a']['end'];
        return $result;
    }

    function send_incoming_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $resp = array();
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                if($xml['end'])
                    $where_end .= ' and date_receive<"' . addslashes($xml['end']) . '"';
                $query = 'select id_sms, date_receive, prefix, originator, phone, text_sms from sms_incoming where id_user="' . $id_user . '" and date_receive>"' . addslashes($xml['start']) . '"' . $where_end;
                $result = $SQL->query($query);
                $num = $SQL->num_rows($result);
                for($i=0; $i<$num; $i++)
                {
                    $id_sms = $SQL->result($result,$i, 'id_sms');
                    $date_receive = $SQL->result($result,$i, 'date_receive');
                    $prefix = $SQL->result($result,$i, 'prefix');
                    $originator = $SQL->result($result,$i, 'originator');
                    $phone = $SQL->result($result,$i, 'phone');
                    $text_sms = $SQL->result($result,$i, 'text_sms');
                    $resp['sms'][] = array('id_sms' => $id_sms, 'date_receive' => $date_receive, 'prefix' => $prefix, 'originator' => $originator, 'phone' => $phone, 'text_sms' => $text_sms);
                }
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }

    function resp_incoming_from_array($answer)
    {
        global $global_languages;
        if(isset($answer['error'])){
            $resp = send_xml_error($answer['error']);
            return $resp;
        }
        if($answer !== FALSE)
        {
            if(is_array($answer['sms']))
            {
                foreach($answer['sms'] as $sms)
                {
                    $resp .= '<sms id_sms="' . htmlspecialchars($sms['id_sms']) . '" date_receive="' . htmlspecialchars($sms['date_receive']) . '" originator="' . htmlspecialchars($sms['originator']) . '" prefix="' . htmlspecialchars($sms['prefix']) . '" phone="' . htmlspecialchars($sms['phone']) . '">' . htmlspecialchars($sms['text_sms']) . '</sms>';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';

        }
        else
            $resp = send_xml_error($global_languages['INCORRECT_LOGIN_OR_PASSWORD']);
        return $resp;
    }

    function get_def_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        foreach($xml['phones'][0]['v']['phone'] as $phone)
        {
            $result['phones'][] = $phone['v'];
        }
        return $result;
    }

    function get_add_originator_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        foreach($xml['originator'][0]['v']['add'] as $originator)
        {
            $result['originator'][] = $originator['v'];
        }
        return $result;
    }

    function send_def_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                if(is_array($xml['phones']))
                {
                    set_phone_code_in_array();
                    foreach($xml['phones'] as $phone)
                    {
                        $phone = get_replace_phone($phone);
                        $res = get_region('', '', $phone);
                        $time_zone = $res['time_zone'];
                        $region = $res['region'];
                        if(!$region)
                        {
                            $region = 'unknown';
                            $time_zone = 'unknown';
                        }

                        if($res['operator'])
                            $operator = $res['operator'];
                        else
                            $operator = 'unknown';
                        $resp[] = array('time_zone' => $time_zone, 'region' => $region, 'operator' => $operator, 'phone' => $phone);
                    }
                }
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }

    function resp_def_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer))
            {
                foreach($answer as $phone)
                {
                    $resp .= '<phone operator="' . htmlspecialchars($phone['operator']) . '" region="' . htmlspecialchars($phone['region']) . '" time_zone="' . htmlspecialchars($phone['time_zone']) . '">' . htmlspecialchars($phone['phone']) . '</phone>';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';

        }
        return $resp;
    }

    function get_list_bases_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        return $result;
    }

    function send_list_bases_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                $query = 'select * from sms_base where id_user="' . $id_user . '"';
                $result = $SQL->query($query);
                $num = $SQL->num_rows($result);
                for($i=0; $i<$num; $i++)
                {
                    $id_base = $SQL->result($result, $i, 'id_base');
                    $name_base = $SQL->result($result, $i, 'name_base');
                    $time_birth = substr($SQL->result($result, $i, 'time_birth'), 0, 5);
                    $day_before = $SQL->result($result, $i, 'day_before');
                    $local_time_birth = get_yes($SQL->result($result, $i, 'local_time_birth'));
                    $originator_birth = $SQL->result($result, $i, 'originator_birth');
                    $text_birth = $SQL->result($result, $i, 'text_birth');
                    $on_birth = get_yes($SQL->result($result, $i, 'on_birth'));

                    $resp[] = array('id_base' => $id_base, 'name_base' => $name_base, 'time_birth' => $time_birth, 'local_time_birth' => $local_time_birth, 'day_before' => $day_before, 'originator_birth' => $originator_birth, 'text_birth' => $text_birth, 'on_birth' => $on_birth);
                }
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }

    function resp_list_bases_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer))
            {
                foreach($answer as $base)
                {
                    $resp .= '<base id_base="' . htmlspecialchars($base['id_base']) . '" name_base="' . htmlspecialchars($base['name_base']) . '" time_birth="' . htmlspecialchars($base['time_birth']) . '" local_time_birth="' . htmlspecialchars($base['local_time_birth']) . '" day_before="' . htmlspecialchars($base['day_before']) . '"  originator_birth="' . htmlspecialchars($base['originator_birth']) . '" on_birth="' . htmlspecialchars($base['on_birth']) . '">' . htmlspecialchars($base['text_birth']) . '</base>';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';

        }
        return $resp;
    }

    function get_list_stop_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        return $result;
    }

    function send_list_stop_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                $query = 'select * from sms_stop where id_user="' . $id_user . '"';
                $result = $SQL->query($query);
                $num = $SQL->num_rows($result);
                for($i=0; $i<$num; $i++)
                {
                    $phone = $SQL->result($result, $i, 'phone');
                    $resp[] = array('phone' => $phone);
                }
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }

    function resp_list_stop_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer))
            {
                foreach($answer as $phone)
                {
                    $resp .= '<phone>' . htmlspecialchars($phone['phone']) . '</phone>';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';

        }
        return $resp;
    }

    function get_bases_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        if(isset($xml['bases'][0]['v']['base']))
        {
           $i=0;
            foreach($xml['bases'][0]['v']['base'] as $base)
            {
                $result['bases'][] = array('id_base' => $base['a']['id_base'], 'number_base' => $base['a']['number_base'], 'name_base' => $base['a']['name_base'], 'time_birth' => $base['a']['time_birth'], 'local_time_birth' => set_yes($base['a']['local_time_birth']), 'day_before' => $base['a']['day_before'], 'originator_birth' => $base['a']['originator_birth'], 'on_birth' => set_yes($base['a']['on_birth']), 'text_birth' => $base['v']);
            }
        }
        if(isset($xml['delete_bases'][0]['v']['base']))
        {
            foreach($xml['delete_bases'][0]['v']['base'] as $base)
            {
                $result['delete_bases'][] = array('id_base' => $base['a']['id_base']);
            }
        }
        return $result;
    }

    function send_bases_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                if(is_array($xml['bases']))
                {
                    foreach($xml['bases'] as $base)
                    {
                        if($base['id_base'])
                        {
                            $query = 'SELECT count(*) as count FROM sms_base where id_base="' . addslashes($base['id_base']) . '" and id_user="' . $id_user . '"';
                            $result = $SQL->query($query);
                            $res = $SQL->fetch_assoc($result);
                            if($res['count']==1)
                            {
                                $query = 'update sms_base set name_base="' . addslashes($base['name_base']) . '", on_birth="' . addslashes($base['on_birth']) . '", time_birth="' . addslashes($base['time_birth']) . '", day_before="' . addslashes($base['day_before']) . '", local_time_birth="' . addslashes($base['local_time_birth']) . '", originator_birth="' . addslashes($base['originator_birth']) . '", text_birth="' . addslashes($base['text_birth']) . '" where id_base="' . addslashes($base['id_base']) . '" and id_user="' . $id_user . '"';
                                $SQL->query($query);
                                if($SQL->affected_rows())
                                    $base['action'] = 'edit';
                                else
                                    $base['action'] = 'not_edit';
                            }else
                                $base['action'] = 'not_found';
                        }
                        else
                        {
                            $query = 'insert into sms_base set name_base="' . addslashes($base['name_base']) . '", on_birth="' . addslashes($base['on_birth']) . '", time_birth="' . addslashes($base['time_birth']) . '", day_before="' . addslashes($base['day_before']) . '", local_time_birth="' . addslashes($base['local_time_birth']) . '", originator_birth="' . addslashes($base['originator_birth']) . '", text_birth="' . addslashes($base['text_birth']) . '", id_user="' . $id_user . '", name_region="' . $global_languages['DEFAULT_NAME_REGION'] . '", name_operator="' . $global_languages['DEFAULT_NAME_OPERATOR'] . '", name_name="' . $global_languages['DEFAULT_NAME_NAME'] . '", name_surname="' . $global_languages['DEFAULT_NAME_SURNAME'] . '", name_patronymic="' . $global_languages['DEFAULT_NAME_PATRONYMIC'] . '", name_date_birth="' . $global_languages['DEFAULT_NAME_DATE_BIRTH'] . '", name_male="' . $global_languages['DEFAULT_NAME_MALE'] . '", name_addition_1="' . $global_languages['DEFAULT_NAME_ADDITION_1'] . '",
                                name_addition_2="' . $global_languages['DEFAULT_NAME_ADDITION_2'] . '", name_addition_3="' . $global_languages['DEFAULT_NAME_ADDITION_2'] . '", name_addition_4="' . $global_languages['DEFAULT_NAME_ADDITION_4'] . '", name_addition_5="' . $global_languages['DEFAULT_NAME_ADDITION_5'] . '"';
                            $SQL->query($query);
                            $base['id_base'] = $SQL->insert_id();
                            $base['action'] = 'insert';
                        }
                        $resp[] = array('id_base' => $base['id_base'], 'number_base' => $base['number_base'], 'action' => $base['action']);
                    }
                }
                if(is_array($xml['delete_bases']))
                {
                    foreach($xml['delete_bases'] as $base)
                    {
                        if($base['id_base'])
                        {
                            $query = 'delete from sms_base where id_base="' . addslashes($base['id_base']) . '" and id_user="' . $id_user . '"';
                            $SQL->query($query);
                            if($SQL->affected_rows())
                                $base['action'] = 'delete';
                            else
                                $base['action'] = 'not_found';
                        }
                        $resp[] = array('id_base' => $base['id_base'], 'action' => $base['action']);
                    }
                }
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }

    function resp_bases_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer))
            {
                foreach($answer as $base)
                {
                    if($base['number_base'])
                        $number_base = ' number_base="' . htmlspecialchars($base['number_base']) . '"';
                    else
                        $number_base = '';
                    $resp .= '<base id_base="' . htmlspecialchars($base['id_base']) . '"' . $number_base . '>' . $base['action'] . '</base>';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';
        }
        return $resp;
    }

    function get_list_phones_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        $result['id_base'] = $xml['base'][0]['a']['id_base'];
        $result['page'] = $xml['base'][0]['a']['page'];
        $result['last_update'] = $xml['base'][0]['a']['last_update'];
        return $result;
    }

    function send_list_phones_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                $query = 'select name_base from sms_base where id_user="' . $id_user . '" and id_base="' . (int)$xml['id_base'] . '"';
                $result = $SQL->query($query);
                if($SQL->num_rows($result))
                {
                    if(!$xml['page'])
                        $xml['page'] = 0;
                    else
                        $xml['page']--;

                    $query = 'select count(*) as num from sms_phone where id_base="' . (int)$xml['id_base'] . '"';
                    $result = $SQL->query($query);
                    $num = $SQL->result($result, 0, 'num');
                    $resp['num_pages'] = ceil($num/XML_NUM_ROWS_PHONE);
                    $resp['page'] = $xml['page']+1;
                    if($xml['last_update'])
                       $where_last_update= ' and last_update>="' . addslashes($xml['last_update']) . '"';
                    else
                       $where_last_update='';

                    $query = 'select * from sms_phone where id_base="' . addslashes($xml['id_base']) . '"'.$where_last_update.' limit ' . ((int)$xml['page']*XML_NUM_ROWS_PHONE) . ', ' . XML_NUM_ROWS_PHONE;
                    $result = $SQL->query($query);
                    $num = $SQL->num_rows($result);
                    for($i=0; $i<$num; $i++)
                    {
                        $phone = $SQL->result($result, $i, 'phone');
                        $region = get_name_region($SQL->result($result, $i, 'id_region'));
                        $operator = get_name_operator($SQL->result($result, $i, 'id_operator'));
                        $name = $SQL->result($result, $i, 'name');
                        $surname = $SQL->result($result, $i, 'surname');
                        $patronymic = $SQL->result($result, $i, 'patronymic');
                        $date_birth = $SQL->result($result, $i, 'date_birth');
                        $male = $SQL->result($result, $i, 'male');
                        $addition_1 = $SQL->result($result, $i, 'addition_1');
                        $addition_2 = $SQL->result($result, $i, 'addition_2');
                        $last_update=$SQL->result($result, $i, 'last_update');
                        $resp['phones'][] = array('phone' => $phone, 'region' => $region, 'operator' => $operator, 'name' => $name, 'surname' => $surname, 'patronymic' => $patronymic, 'date_birth' => $date_birth, 'male' => $male, 'addition_1' => $addition_1, 'addition_2' => $addition_2,'last_update' => $last_update);
                    }
                }
                else
                    $resp['error'] = $global_languages['BASE_NAME_NOT_EXIST'];
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }

    function resp_list_phones_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer['phones']))
            {
                foreach($answer['phones'] as $phone)
                {
                    $resp .= '<phone phone="' . htmlspecialchars($phone['phone']) . '" region="' . htmlspecialchars($phone['region']) . '" operator="' . htmlspecialchars($phone['operator']) . '" name="' . htmlspecialchars($phone['name']) . '" surname="' . htmlspecialchars($phone['surname']) . '" patronymic="' . htmlspecialchars($phone['patronymic']) . '" date_birth="' . htmlspecialchars($phone['date_birth']) . '" male="' . htmlspecialchars($phone['male']) . '" addition_1="' . htmlspecialchars($phone['addition_1']) . '" addition_2="' . htmlspecialchars($phone['addition_2']) . '" last_update="' . htmlspecialchars($phone['last_update']) . '"/>';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    <phones page="' . htmlspecialchars($answer['page']) . '" num_pages="' . htmlspecialchars($answer['num_pages']) . '">
        ' . $resp . '
    </phones>
</response>';

        }
        return $resp;
    }

    function get_phones_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        $result['id_base'] = $xml['base'][0]['a']['id_base'];
        foreach($xml['base'][0]['v']['phone'] as $phone)
        {
            $result['phones'][] = array('phone' => $phone['a']['phone'], 'number_phone' => $phone['a']['number_phone'], 'action' => $phone['a']['action'], 'region' => $phone['a']['region'], 'operator' => $phone['a']['operator'], 'name' => $phone['a']['name'], 'surname' => $phone['a']['surname'], 'patronymic' => $phone['a']['patronymic'], 'date_birth' => $phone['a']['date_birth'], 'male' => $phone['a']['male'], 'addition_1' => $phone['a']['addition_1'], 'addition_2' => $phone['a']['addition_2']);
        }
        return $result;
    }

    function send_phones_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                $query = 'select name_base from sms_base where id_user="' . $id_user . '" and id_base="' . (int)$xml['id_base'] . '"';
                $result = $SQL->query($query);
                if($SQL->num_rows($result))
                {
                    if(is_array($xml['phones']))
                    {
                        set_phone_code_in_array();
                        foreach($xml['phones'] as $phone)
                        {
                            $phone['phone'] = (int)get_replace_phone($phone['phone']);
                            if($phone['phone'])
                            {
                                if($phone['action'] == 'delete')
                                {
                                    $query = 'delete from sms_phone where id_base="' . (int)$xml['id_base'] . '" and phone="' . $phone['phone'] . '"';
                                    $SQL->query($query);
                                    if($SQL->affected_rows())
                                        $action = 'delete';
                                    else
                                        $action = 'not_found';
                                }
                                else
                                {
                                    $def = get_region($phone['operator'], $phone['region'], $phone['phone']);
                                    $set = '';
                                    if($phone['name'])
                                        $set .= ', name="' . addslashes($phone['name']) . '"';
                                    if($phone['surname'])
                                        $set .= ', surname="' . addslashes($phone['surname']) . '"';
                                    if($phone['patronymic'])
                                        $set .= ', patronymic="' . addslashes($phone['patronymic']) . '"';
                                    if($phone['date_birth'])
                                        $set .= ', date_birth="' . addslashes($phone['date_birth']) . '"';
                                    $phone['male'] = mb_strtolower($phone['male']);
                                    if(!$phone['male'])
                                        $phone['male'] = 'NULL';
                                    elseif(strrpos($phone['male'], 'м') !== FALSE)
                                        $phone['male'] = '"m"';
                                    else
                                        $phone['male'] = '"f"';
                                    if($phone['male'])
                                        $set .= ', male=' . $phone['male'];
                                    if($phone['addition_1'])
                                        $set .= ', addition_1="' . addslashes($phone['addition_1']) . '"';
                                    if($phone['addition_2'])
                                        $set .= ', addition_2="' . addslashes($phone['addition_2']) . '"';

                                    $query = 'select id_phone from sms_phone where id_base="' . (int)$xml['id_base'] . '" and phone="' . $phone['phone'] . '"';
                                    $result = $SQL->query($query);
                                    if($SQL->num_rows($result))
                                    {
                                        $query = 'update sms_phone set id_region="' . $def['id_region'] . '", id_operator="' . $def['id_operator'] . '", MCC="' . $def['MCC'] . '", MNC="' . $def['MNC'] . '"' . $set . ' where id_base="' . addslashes($xml['id_base']) . '" and phone="' . $phone['phone'] . '"';
                                        $SQL->query($query);
                                        $action = 'edit';
                                    } else {
                                        $query = 'insert into sms_phone set id_base="' . (int)$xml['id_base'] . '", phone="' . $phone['phone'] . '", id_region="' . $def['id_region'] . '", id_operator="' . $def['id_operator'] . '", MCC="' . $def['MCC'] . '", MNC="' . $def['MNC'] . '"' . $set;
                                        @$SQL->query($query);
                                        $action = 'insert';
                                    }
                                }
                                $resp['phones'][] = array('phone' => $phone['phone'], 'number_phone' => $phone['number_phone'], 'action' => $action);
                            }
                        }
                    }
                    $resp['id_base'] = (int)$xml['id_base'];
                }
                else
                    $resp['error'] = $global_languages['BASE_NAME_NOT_EXIST'];
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }

    function resp_phones_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer['phones']))
            {
                foreach($answer['phones'] as $phone)
                {
                    $resp .= '<phone phone="' . htmlspecialchars($phone['phone']) . '" number_phone="' . htmlspecialchars($phone['number_phone']) . '">' . $phone['action'] . '</phone>';
                }
            }
            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response><base id_base="' . $answer['id_base'] . '">' . $resp . '</base></response>';
        }
        return $resp;
    }


    function get_stop_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        if(isset($xml['add_stop'][0]['v']['phone']))
        {
            foreach($xml['add_stop'][0]['v']['phone'] as $phone)
            {
                $result['add_stop'][] = array('phone' => $phone['a']['phone']);
            }
        }
        if(isset($xml['delete_stop'][0]['v']['phone']))
        {
            foreach($xml['delete_stop'][0]['v']['phone'] as $phone)
            {
                $result['delete_stop'][] = array('phone' => $phone['a']['phone']);
            }
        }

        return $result;
    }

    function send_stop_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                if(is_array($xml['add_stop']))
                {
                    foreach($xml['add_stop'] as $phone)
                    {
                        $phone['phone'] = (int) get_phone($phone['phone']);
                        $query = 'insert into sms_stop set id_user="' . $id_user . '", phone="' . $phone['phone'] . '"';
                        $SQL->query($query);
                        $phone['action'] = 'add';
                        $resp[] = array('phone' => $phone['phone'], 'action' => $phone['action']);
                    }
                }
                if(is_array($xml['delete_stop']))
                {
                    foreach($xml['delete_stop'] as $phone)
                    {
                        $phone['phone'] = (int) get_phone($phone['phone']);
                        $query = 'delete from sms_stop where id_user="' . $id_user . '" and phone="' . $phone['phone'] . '"';
                        $SQL->query($query);
                        if($SQL->affected_rows())
                            $phone['action'] = 'delete';
                        else
                            $phone['action'] = 'not_found';
                        $resp[] = array('phone' => $phone['phone'], 'action' => $phone['action']);
                    }
                }
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }

    function resp_stop_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer))
            {
                foreach($answer as $phone)
                {
                    $resp .= '<phone phone="' . htmlspecialchars($phone['phone']) . '">' . $phone['action'] . '</phone>';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';
        }
        return $resp;
    }

    function get_list_scheduled_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        $result['page'] = $xml['scheduled'][0]['a']['page'];
        return $result;
    }

    function send_list_scheduled_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            if(!$xml['page'])
                $xml['page'] = 0;
            else
                $xml['page']--;

            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                $query = 'select count(*) as num from sms_turn_time where id_user="' . $id_user . '"';
                $result = $SQL->query($query);
                $num = $SQL->result($result, 0, 'num');
                $resp['num_pages'] = ceil($num/XML_NUM_ROWS_PHONE);
                $resp['page'] = $xml['page']+1;

                $query = 'select * from sms_turn_time where id_user="' . $id_user . '" limit ' . ((int)$xml['page']*XML_NUM_ROWS_PHONE) . ', ' . XML_NUM_ROWS_PHONE;
                $result = $SQL->query($query);
                $num = $SQL->num_rows($result);
                for($i=0; $i<$num; $i++)
                {
                    $id_turn = $SQL->result($result, $i, 'id_turn');
                    $time_put_turn = $SQL->result($result, $i, 'time_put_turn');
                    $originator = $SQL->result($result, $i, 'originator');
                    $phone = $SQL->result($result, $i, 'phone');
                    $type_sms = $SQL->result($result, $i, 'type_sms');
                    $text_sms = $SQL->result($result, $i, 'text_sms');
                    $count_sms = $SQL->result($result, $i, 'count_sms');
                    $sms_priority = $SQL->result($result, $i, 'sms_priority');
                    $name_delivery = get_name_delivery($SQL->result($result, $i, 'id_name_delivery'));
                    $send_from = $SQL->result($result, $i, 'send_from');
                    $send_from_date = $SQL->result($result, $i, 'send_from_date');
                    $time_send = $send_from_date . ' ' . substr($send_from, 0, 5);
                    $validity_period = $SQL->result($result, $i, 'validity_period');

                    $resp['scheduled'][] = array('id_sms' => $id_turn, 'time_put_turn' => $time_put_turn, 'originator' => $originator, 'phone' => $phone, 'type_sms' => $type_sms, 'text_sms' => $text_sms, 'count_sms' => $count_sms, 'sms_priority' => $sms_priority, 'name_delivery' => $name_delivery, 'time_send' => $time_send, 'validity_period' => $validity_period);
                }
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }

    function resp_list_scheduled_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer['scheduled']))
            {
                foreach($answer['scheduled'] as $scheduled)
                {
                    $resp .= '<scheduled id_sms="' . htmlspecialchars($scheduled['id_sms']) . '" time_put_turn="' . htmlspecialchars($scheduled['time_put_turn']) . '" originator="' . htmlspecialchars($scheduled['originator']) . '" phone="' . htmlspecialchars($scheduled['phone']) . '" type_sms="' . htmlspecialchars($scheduled['type_sms']) . '" text_sms="' . htmlspecialchars($scheduled['text_sms']) . '" count_sms="' . htmlspecialchars($scheduled['count_sms']) . '" name_delivery="' . htmlspecialchars($scheduled['name_delivery']) . '" time_send="' . htmlspecialchars($scheduled['time_send']) . '" validity_period="' . htmlspecialchars($scheduled['validity_period']) . '" />';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    <phones page="' . htmlspecialchars($answer['page']) . '" num_pages="' . htmlspecialchars($answer['num_pages']) . '">
        ' . $resp . '
    </phones>
</response>';

        }
        return $resp;
    }

    function get_scheduled_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        foreach($xml['delete_schedule'][0]['v']['schedule'] as $scheduled)
        {
            $result['delete_scheduled'][] = array('id_sms' => $scheduled['a']['id_sms']);
        }
        return $result;
    }

    function send_scheduled_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                if(is_array($xml['delete_scheduled']))
                {
                    foreach($xml['delete_scheduled'] as $scheduled)
                    {
                        $count_delete_sms = return_sms((int)$scheduled['id_sms'], 'sms_turn_time', $id_user);
                        if($count_delete_sms)
                            $action = 'delete';
                        else
                            $action = 'not_found';
                        $resp[] = array('id_sms' => (int)$scheduled['id_sms'], 'action' => $action);
                    }
                }
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }

    function resp_scheduled_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer))
            {
                foreach($answer as $scheduled)
                {
                    $resp .= '<scheduled id_sms="' . htmlspecialchars($scheduled['id_sms']) . '">' . $scheduled['action'] . '</scheduled>';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';
        }
        return $resp;
    }

    function get_check_change_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        $result['obgect'] = $xml['check'][0]['a']['obgect'];
        $result['id'] = $xml['check'][0]['a']['id'];
        return $result;
    }

    function send_check_change_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                if($result['obgect'] == 'base')
                {
                    $query = 'select * from sms_base where id_user="' . $id_user . '" and id_base="' . (int)$xml['id'] . '"';
                    $result = $SQL->query($query);
                    if($SQL->num_rows($result))
                        $resp['error'] = $global_languages['BASE_NAME_NOT_EXIST'];
                }
                if(!isset($resp['error']))
                {
                    $query = 'select * from sms_update where type="' . addslashes($xml['obgect']) . '" and id="' . (int)$xml['id'] . '" order by time_update desc limit 0, 1';
                    $result = $SQL->query($query);
                    if($SQL->num_rows($result))
                        $time_update = $SQL->result($result, $i, 'time_update');
                    else
                        $time_update = '0000-00-00 00:00:00';
                    $resp[] = array('time_update' => $time_update);
                }
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }

    function resp_check_change_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer))
            {
                foreach($answer as $base)
                {
                    $resp .= '<obgect time_update="' . $base['time_update'] . '" />';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';

        }
        return $resp;
    }

    function get_name_region($id_region)
    {
        global $SQL;
        $query = 'select region from sms_phone_region where id_region="' . $id_region . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
            return $SQL->result($result, 0, 'region');
        else
           return FALSE;
    }

    function get_name_operator($id_operator)
    {
        global $SQL;
        $query = 'select operator from sms_phone_operator where id_operator="' . $id_operator . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
            return $SQL->result($result, 0, 'operator');
        else
           return FALSE;
    }

    function get_yes($v)
    {
        if($v)
            $v = 'yes';
        else
            $v = 'no';
        return $v;
    }

    function set_yes($v)
    {
        if($v == 'yes')
            $v = 1;
        else
            $v = 0;
        return $v;
    }

    function check_user_e_mail($e_mail, $id_user)
    {
        global $SQL;
        if($e_mail)
        {
            $e_mail = strtolower($e_mail);
            $query = 'select id_user from sms_e_mail where id_user="' . $id_user . '" and e_mail="' . addslashes($e_mail) . '"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
                return TRUE;
            else
            {
                $query = 'select id_user from sms_e_mail where id_user="' . $id_user . '"';
                $result = $SQL->query($query);
                if($SQL->num_rows($result))
                    return FALSE;
                else
                    return TRUE;
            }
        }
        else
            return TRUE;
    }

    function send_error($str, $num_message)
    {
        $str = '<information number_sms="' . $num_message . '">' . $str . '</information>
';
        return $str;
    }

    function send_xml($src,$href)
    {
        $res = '';

$src = str_replace("
", '', $src);

        $ch = curl_init();
        curl_setopt($ch, CURLOPT_HTTPHEADER, array('Content-type: text/xml; charset=utf-8'));
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_CRLF, true);
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $src);
        curl_setopt($ch, CURLOPT_URL, $href);
        $result = curl_exec($ch);

        $res = $result;

        curl_close($ch);

        return $res;
    }

    function send_state_xml($id_state, $time_change_state, $state, $url_get_state)
    {
        $src = '<?xml version="1.0" encoding="utf-8"?><request><state id_sms="' . $id_state . '" time="' . $time_change_state . '">' . $state . '</state></request>';

        $res=send_xml($src,$url_get_state);

        if(strpos($res, $id_state) === FALSE)
            return FALSE;
        else
            return TRUE;
    }
    // Для синхронизации баланса
    function get_balance_xml($login, $password, $url)
    {
        $src = '<?xml version="1.0" encoding="utf-8"?><request><security><login value="'.$login.'"/><password value="'.$password.'"/></security></request>';

        $res=send_xml($src,$url);

        $xml=XML_in_array($src);
        if($xml === FALSE)
            return FALSE;
        foreach($balance=$xml['sms'] as $sms)
        {
            if($sms['a']['area']=='Россия')
            {
                $balance=$sms['v'];
                break;
            }else
                $balance=FALSE;
        }

        if(is_numeric($balance))
            return $balance-50;
        return FALSE;
    }




    // Запись окончательного статуса
    function set_final_state($state, $command_status, $replace_partly_deliver, $id_sms, $id_message, $part_no, $id_aggregating, $time_change_state = '', $id_smpp_error = 0)
    {
        global $SQL, $query_log;
       if($id_message || $id_sms)
       {
            if($id_sms)
            {
                $where ='id_sms="' . $id_sms . '"';
                if($part_no)
                    $where .=' and part_no="' . $part_no . '"';
                $set_state = ', id_aggregating="' . $id_aggregating . '"';
            }
            else
            {
                $where ='id_part_operator="' . $id_message . '" and id_aggregating="' . $id_aggregating.'"';
                $set_state = '';
            }
            if($time_change_state)
                $time_change_state = ', time_change_state="' . $time_change_state . '"';
            else
                $time_change_state = ', time_change_state=CURRENT_TIMESTAMP';
              $query = 'select id_state, id_user, id_sms, final, part_no, sync, send, send_xml, sell_we , buy_we, sell_partner , buy_partner, id_currency, id_currency_aggregating, deliver_payment_sell_we, deliver_payment_buy_we, deliver_payment_sell_partner, deliver_payment_buy_partner from sms_state_final where '.$where;
              $query_log=$query;
              $result = $SQL->query($query);
              $num = $SQL->num_rows($result);

              if($num)
              {
                 $id_sms_array=array();
                 for($i=0; $i<$num; $i++)
                 {
                    $final = $SQL->result($result, $i, 'final');
                    if(!$final)
                    {
                        $id_sms = $SQL->result($result, $i, 'id_sms');
                        $id_sms_array[$id_sms]['count']++;
                        $id_sms_array[$id_sms]['id_currency'] = $SQL->result($result, $i, 'id_currency');
                        $id_sms_array[$id_sms]['id_currency_aggregating'] = $SQL->result($result, $i, 'id_currency_aggregating');

                        $sell_we = $SQL->result($result, $i, 'sell_we');
                        $buy_we = $SQL->result($result, $i, 'buy_we');
                        $sell_partner = $SQL->result($result, $i, 'sell_partner');
                        $buy_partner = $SQL->result($result, $i, 'buy_partner');

                        $id_sms_array[$id_sms]['sell_we'] += $sell_we;
                        $id_sms_array[$id_sms]['buy_we'] += $buy_we;
                        $id_sms_array[$id_sms]['sell_partner'] += $sell_partner;
                        $id_sms_array[$id_sms]['buy_partner'] += $buy_partner;

                        if($state == 'not_deliver' || $state == 'expired' || $state == 'partly_deliver')
                        {
                            if($deliver_payment_sell_we = $SQL->result($result, $i, 'deliver_payment_sell_we'))
                                $repayment_sell_we = $sell_we;
                            if($deliver_payment_buy_we = $SQL->result($result, $i, 'deliver_payment_buy_we'))
                                $repayment_buy_we = $buy_we;
                            if($deliver_payment_sell_partner = $SQL->result($result, $i, 'deliver_payment_sell_partner'))
                                $repayment_sell_partner = $sell_partner;
                            if($deliver_payment_buy_partner = $SQL->result($result, $i, 'deliver_payment_buy_partner'))
                                $repayment_buy_partner = $buy_partner;

                            $id_sms_array[$id_sms]['deliver_payment_sell_we'] += $deliver_payment_sell_we;
                            $id_sms_array[$id_sms]['deliver_payment_buy_we'] += $deliver_payment_buy_we;
                            $id_sms_array[$id_sms]['deliver_payment_sell_partner'] += $deliver_payment_sell_partner;
                            $id_sms_array[$id_sms]['deliver_payment_buy_partner'] += $deliver_payment_buy_partner;
                        }
                        else
                        {
                            $repayment_sell_we = 0;
                            $repayment_buy_we = 0;
                            $repayment_sell_partner = 0;
                            $repayment_buy_partner = 0;
                        }

                        $id_sms_array[$id_sms]['states'][] = array('id_state'=>$SQL->result($result, $i, 'id_state'), 'part_no' => $SQL->result($result, $i, 'part_no'),'sync' => $SQL->result($result, $i, 'sync'), 'send' => $SQL->result($result, $i, 'send'), 'send_xml' => $SQL->result($result, $i, 'send_xml'), 'repayment_sell_we' => $repayment_sell_we, 'repayment_buy_we' => $repayment_buy_we, 'repayment_sell_partner' => $repayment_sell_partner, 'repayment_buy_partner' => $repayment_buy_partner);
                        $id_sms_array[$id_sms]['repayment_sell_we'] += $repayment_sell_we;
                        $id_sms_array[$id_sms]['repayment_buy_we'] += $repayment_buy_we;
                        $id_sms_array[$id_sms]['repayment_sell_partner'] += $repayment_sell_partner;
                        $id_sms_array[$id_sms]['repayment_buy_partner'] += $repayment_buy_partner;
                    }
                 }
                 if(is_array($id_sms_array))
                 {
                     foreach ($id_sms_array as $id_sms=>$params)
                     {

                         $query = 'select id_partner, id_package_partner, MCC, MNC, MNC_user, MNC_partner, time, phone, originator, id_name_delivery,num_parts, id_user, id_package_user from sms_send_trafic where id_sms="' . $id_sms . '"';
                         $result_trafic = $SQL->query($query);

                         if($SQL->num_rows($result_trafic))
                         {
                             $id_user = $SQL->result($result_trafic, 0, 'id_user');
                             $id_package_user = $SQL->result($result_trafic, 0, 'id_package_user');
                             $id_partner = $SQL->result($result_trafic, 0, 'id_partner');
                             $id_package_partner = $SQL->result($result_trafic, 0, 'id_package_partner');
                             $MCC = $SQL->result($result_trafic, 0, 'MCC');
                             $MNC = $SQL->result($result_trafic, 0, 'MNC');
                             $MNC_user = $SQL->result($result_trafic, 0, 'MNC_user');
                             $MNC_partner = $SQL->result($result_trafic, 0, 'MNC_partner');
                             $phone = $SQL->result($result_trafic, 0, 'phone');
                             $time = $SQL->result($result_trafic, 0, 'time');
                             $originator = $SQL->result($result_trafic, 0, 'originator');
                             $id_name_delivery = $SQL->result($result_trafic, 0, 'id_name_delivery');
                             $num_parts = $SQL->result($result_trafic, 0, 'num_parts');

                             if($state == 'not_deliver' || $state == 'expired' || $state == 'partly_deliver')
                                return_balance_param($id_package_user, $id_package_partner, $params['repayment_sell_we'], $params['repayment_buy_we'], $params['repayment_sell_partner'], $params['deliver_payment_sell_we'], $params['deliver_payment_buy_we'], $params['deliver_payment_sell_partner']);// Возращ балан

                            foreach ($params['states'] as $param)
                            {

                                 if($param['send'])
                                  {
                                      $query = 'insert into sms_state_smpp set id_state="' . $param['id_state'] . '", id_user="' . $id_user . '", id_sms="' . $id_sms . '", time_change_state=CURRENT_TIMESTAMP, state="' . $state . '", part_no="' . $param['part_no'] . '", id_smpp_error="' . $id_smpp_error . '"';
                                      $SQL->query_async($query);
                                  }
                                  if($param['send_xml'])
                                  {
                                        $query = 'insert into sms_state_xml set id_state="' . $param['id_state'] . '", id_user="' . $id_user . '", id_sms="' . $id_sms . '", state="' . $state . '", part_no="' . $param['part_no'] . '"';
                                       $SQL->query_async($query);
                                  }
                                  if($state!='partly_deliver')
                                  {
                                    if($state == 'not_deliver' && $id_smpp_error)
                                    {
                                        $query = 'update sms_phone_error set count=count+1 where phone="' . $phone . '" and id_smpp_error="' . $id_smpp_error . '"';
                                        $SQL->query($query);
                                        if($SQL->affected_rows() == 0)
                                        {
                                             $query = 'insert into sms_phone_error set phone="' . $phone . '", id_smpp_error="' . $id_smpp_error . '", count=1';
                                             $SQL->query_async($query);
                                        }
                                    }
                                    $query = 'update sms_phone_state set num_' . $state . '=num_' . $state . '+1 where phone="' . $phone . '"';
                                    $SQL->query($query);
                                    if($SQL->affected_rows() == 0)
                                    {
                                        $query = 'insert into sms_phone_state set phone="' . $phone . '", num_' . $state . '=1';
                                        $SQL->query_async($query);
                                    }
                                  }
                                $query = 'update sms_state_final  set state="' . $state . '",final="1", command_status="'.$command_status.'" , id_smpp_error="' . $id_smpp_error . '"' . $time_change_state .', repayment_sell_we="' . $param['repayment_sell_we']. '", repayment_buy_we="' . $param['repayment_buy_we'] . '", repayment_sell_partner="' . $param['repayment_sell_partner'] . '", repayment_buy_partner="' . $param['repayment_buy_partner'] . '"' . $set_state . ' where id_state="' . $param['id_state'] . '"';
                                $SQL->query_async($query);
                                if($state!='partly_deliver' && $state!='expired' && $id_message && $id_aggregating && $id_sms){
                                    $query_check = "SELECT tmp.id_sms FROM `sms_state_final` as tmp  WHERE `id_sms`='{$id_sms}' AND tmp.final='1'";
                                    $result_check = $SQL->query($query_check);
                                    $query_log=$SQL->num_rows($result_check);
                                    if($SQL->num_rows($result_check)==$num_parts)
                                    {
                                        $query_check2 = "SELECT tmp.id_sms FROM `sms_state_final` as tmp  WHERE `id_sms`='{$id_sms}' AND (tmp.state='not_deliver' OR tmp.state='expired')";
                                        $result_check2 = $SQL->query($query_check2);
                                        $query_log=$query_check2;
                                        $time_ = substr($time, 0, 10) . ' 00:00:00';
                                        if($SQL->num_rows($result_check2))
                                        {
                                            $state_ = 'not_deliver';
                                            $query = 'update sms_phone_state2 set num_' . $state_ . '=num_' . $state_ . '+1, time_last_' . $state_ . '="' . $time_ . '" where phone="' . $phone . '"';
                                            $query_log=$query;
                                            $SQL->query($query);
                                            if($SQL->affected_rows() == 0)
                                            {
                                                $query_log=$query;
                                                $query = 'insert into sms_phone_state2 set phone="' . $phone . '", num_' . $state_ . '=1, time_last_' . $state_ . '="' . $time_ . '"';
                                                $SQL->query_async($query);
                                            }
                                        }else{
                                            $state_ = 'deliver';
                                            $query = 'update sms_phone_state2 set num_' . $state_ . '=num_' . $state_ . '+1, time_last_' . $state_ . '="' . $time_ . '" where phone="' . $phone . '"';
                                            $query_log=$query;
                                            $SQL->query($query);
                                            if($SQL->affected_rows() == 0)
                                            {
                                                $query_log=$query;
                                                $query = 'insert into sms_phone_state2 set phone="' . $phone . '", num_' . $state_ . '=1, time_last_' . $state_ . '="' . $time_ . '"';
                                                $SQL->query_async($query);
                                            }
                                        }
                                    }             
                                        
                                }
                            }
                            set_sms_stat($id_user, $id_partner, $MCC, $MNC, $id_name_delivery, $originator, $state, $id_aggregating, $params['count'], $params['id_currency'], $params['id_currency_aggregating'], $params['sell_we'], $params['buy_we'], $params['sell_partner'], $params['buy_partner'], $params['repayment_sell_we'], $params['repayment_buy_we'], $params['repayment_sell_partner'], $params['repayment_buy_partner'], substr($time, 0, 13) . ':00:00', $replace_partly_deliver);
                         }
                      }
                  }
              }
        }
        if(is_array($id_sms_array))
            return TRUE;
        elseif($num)
            return 'not_final';
        else
            return 'have_not_state';
    }

    function return_balance_param($id_package_user, $id_package_partner, $repayment_sell_we, $repayment_buy_we, $repayment_sell_partner, $deliver_payment_sell_we, $deliver_payment_buy_we, $deliver_payment_sell_partner)  // Возращ баланс
    {
        global $SQL;
        $repayment_sell_we_=$repayment_sell_we;
        $deliver_payment_user = $deliver_payment_sell_we;
        if($id_package_partner)
        {
            if($deliver_payment_sell_we)
            {
                $query = 'update sms_package_partner set waste=waste-'.$repayment_sell_we.' where id_package_partner="' . $id_package_partner . '"';
                $SQL->query_async($query);
            }
            $repayment_sell_we_=$repayment_sell_partner;
            $deliver_payment_user = $deliver_payment_sell_partner;
        }

        if($deliver_payment_user)
        {
            $query = 'update sms_package_user set waste=waste-'.$repayment_sell_we_.' where id_package_user="' . $id_package_user . '"';
            $SQL->query_async($query);
        }
    }

    function mb_get_ord($char)
    {
       $len = strlen($char);
       $code=0;
       for($i=0;$i<$len;$i++)
       {
           $code += pow(0x100,$len-1-$i)*ord($char[$i]);
       }
       return $code;
    }

    function mb_get_char($ord)
    {
        if($ord==0)
            return chr(0);
        $char='';
        while($ord)
        {
            $rest=$ord%0x100;
            $char=chr($rest).$char;
            $ord=($ord-$rest)/0x100;
        }
        return $char;
    }

    function get_char_message($_message)
    {
        global $UTF8_GSM7,$UTF8_ISO_8859_1;
        $len = mb_strlen($_message);
        $not_GSM7 = 0;
        $not_ISO = 0;
        for($i=0;$i<$len;$i++)
        {
            $k=mb_get_ord(mb_substr($_message,$i,1));
            if(!isset($UTF8_ISO_8859_1[$k]) && !isset($UTF8_GSM7[$k]))
                return 0x8;
            elseif(!isset($UTF8_GSM7[$k]))
                   $not_GSM7++;
            elseif(!isset($UTF8_ISO_8859_1[$k]))
                   $not_ISO++;
        }
        if($not_GSM7>$not_ISO)
            return 0x3;
        else
            return 0x0;
    }
    function check_GSM7($string)
    {
        global $UTF8_GSM7;
        $len = strlen($string);
        for($i=0; $i<$len; $i++)
        {
            $char1 = ord($string[$i]);
            if($i < $len-1)
                $char2 = $char1 * 0x100 + ord($string[$i+1]);
            if($i < $len-2)
                $char3 = $char2 * 0x100 + ord($string[$i+2]);

            if($i < $len-2 && isset($UTF8_GSM7[$char3]) && $char1)
            {
                $tmp = $UTF8_GSM7[$char3];
                $i+=2;
            }
            else
            {
                if($i < $len-1 && isset($UTF8_GSM7[$char2]) && $char1)
                {
                    $tmp = $UTF8_GSM7[$char2];
                    $i++;
                }
                elseif(isset($UTF8_GSM7[$char1]))
                   $tmp = $UTF8_GSM7[$char1];
                else
                    $tmp = -0x01;
            }

            if($tmp < 0)
            {
                return FALSE;
            }
        }
        return TRUE;
    }

    function convert_encoding($string, $charset=0x0 ,$to = TRUE)
    {
        if($charset===0x0)
        {
            global $UTF8_GSM7;
            $conv = $UTF8_GSM7;
        }
        else
        {
            global $UTF8_ISO_8859_1;
            $conv = $UTF8_ISO_8859_1;
        }
        if(!$to)
            $conv = array_flip($conv);
        $len = strlen($string);
        $return = '';
        for($i=0;$i<$len;$i++)
        {
            $char1 = ord($string[$i]);
            if($i < $len-1)
                $char2 = $char1 * 0x100 + ord($string[$i+1]);
            if($i < $len-2)
                $char3 = $char2 * 0x100 + ord($string[$i+2]);
            if($i < $len-2 && isset($conv[$char3]) && $char1)
            {
                $tmp = $conv[$char3];
                $i+=2;
            }
            elseif($i < $len-1 && isset($conv[$char2]) && $char1)
            {
                $tmp = $conv[$char2];
                $i+=1;
            }
            elseif(isset($conv[$char1]))
                $tmp = $conv[$char1];
            else
                $tmp = -0x01;
            if($tmp>=0)
                $return .= mb_get_char($tmp);
        }
        return $return;
    }

    function stop_script()
    {
        global $stop_script;
        $stop_script = TRUE;
    }

    function log_to_archive($dir)
    {
        if($handle = opendir($dir))
        {
            $date = date('Y-m-d');
            while(($file = readdir($handle)) !== FALSE)
            {
                if($file != '.' && $file != '..')
                {
                    $last_update = filemtime($dir.'/'.$file);
                    if(is_dir($dir . '/' . $file))
                        log_to_archive($dir . '/' . $file);
                    elseif((($type_file=substr($file, -4)) == '.bz2' || $type_file == '.log') && $last_update<time()-DAY_TO_DELETE_LOG*86400)
                    {
                            unlink($dir.'/'.$file);
                    }elseif($type_file == '.log' && stristr($file, $date) === FALSE)
                    {
                        $cmd = 'nice -n+20 bzip2 "' . $dir . '/' . $file . '"';
                        exec($cmd, $res);
                    }
                }
            }
            closedir($handle);
        }
    }

    function send_from_ftp($log, $dir)
    {
        if($handle = opendir($dir))
        {
            while(($file = readdir($handle)) !== FALSE)
            {
                if($file != '.' && $file != '..')
                {
                    if(is_dir($dir . $file) && is_dir($dir . $file . '/in') && $handle_2 = opendir($dir . $file . '/in'))
                    {
                        while(($file_2 = readdir($handle_2)) !== FALSE)
                        {
                            if($file_2 != '.' && $file_2 != '..' && !is_dir($dir . $file . '/in' . $file_2) && substr($file_2, -4) == '.xml')
                            {
                                send_xml_from_file($log, $dir . $file . '/',  $file_2);
                            }
                        }
                    }
                }
            }
            closedir($handle);
        }
    }

    function get_message_from_get($get)
    {
        $result['login'] = $get['user'];
        $result['password'] = $get['pwd'];
        $result['message'][0]['sender'] = $get['sadr'];
        $result['message'][0]['name_delivery'] = $get['name_delivery'];
        $result['message'][0]['text'] = $get['text'];
        $result['message'][0]['type'] = 'sms';

        $phone_array = get_array_phone($get['dadr']);
        $count_phone = count($phone_array);
        for($i=0, $j=0; $i< $count_phone; $i++)
        {
            if($phone_ = get_phone($phone_array[$i]))
            {
                $result['message'][0]['phones'][$j]['phone'] = $phone_;
                $j++;
            }
        }
        if($j)
            return $result;
        else
            return FALSE;
    }

    function get_message_from_mail($in, $e_mail)
    {
        $tag = explode("\r\n", $in);
        $count_tag = count($tag);
        /*if($count_tag==1){
            $tag2 = explode("<br />", $tag[0]);
            $count_tag2 = count($tag2);
            if($count_tag2>1){
                $tag = $tag2;
                $count_tag = count($tag);
                foreach($tag as $key=>$val){
                    $tag[$key] = strip_tags($val);
                }
            }
        }*/
        $count_papam = 1;
        for($i=0; $i<$count_tag; $i++)
        {
            if(substr($tag[$i], 0, 7) == 'SmsUser')
            {
                $tmp = explode('=', $tag[$i], 2);
                $result['login'] = trim($tmp[1]);
                $count_papam++;
            }
            elseif(substr($tag[$i], 0, 8) == 'Password')
            {
                $tmp = explode('=', $tag[$i], 2);
                $result['password'] = trim($tmp[1]);
                $count_papam++;
            }
            elseif(substr($tag[$i], 0, 13) == 'SourceAddress')
            {
                $tmp = explode('=', $tag[$i], 2);
                $result['message'][0]['sender'] = trim($tmp[1]);
                $count_papam++;
            }
            elseif(substr($tag[$i], 0, 11) == 'PhoneNumber')
            {
                $tmp = explode('=', $tag[$i], 2);
                $phone = trim($tmp[1]);
                $count_papam++;
            }
            elseif($count_papam > 4)
            {
                if($count_papam > 5 && $i<$count_tag-1)
                    $result['message'][0]['text'] .= "\n";
                $result['message'][0]['text'] .= $tag[$i];
                $count_papam++;
            }
        }
        
        $phone_array = get_array_phone($phone);
        $count_phone = count($phone_array);
        for($i=0, $j=0; $i< $count_phone; $i++)
        {
            if($phone_ = get_phone($phone_array[$i]))
            {
                $result['message'][0]['phones'][$j]['phone'] = $phone_;
                $j++;
            }
        }
        $result['e_mail'] = $e_mail;
        $result['message'][0]['type'] = 'sms';
        if($j)
            return $result;
        else
            return FALSE;
    }

    function send_from_mail($log)
    {
        $param_site = get_param_site(0);
        $pop3 = new Net_POP3();
        $pop3->supportedAuthMethods[1] = $param_site['E_MAIL_TO_SMS_AUTH'];
//        $pop3->_debug = TRUE;
        if($pop3->connect($param_site['E_MAIL_TO_SMS_SERVER'], $param_site['E_MAIL_TO_SMS_PORT']))
        {
            if($pop3->login($param_site['E_MAIL_TO_SMS_LOGIN'], $param_site['E_MAIL_TO_SMS_PASSWORD']))
            {
                $num_mail = $pop3->numMsg();
                $log->get_data("num_mail -> " . $num_mail);
                for($i=1; $i<=$num_mail; $i++)
                {
                    $headers = $pop3->getParsedHeaders($i);
                    $log->get_data("headers -> " . $headers);
                    print_r($headers);
                    $e_mail = substr($headers['Return-path'], 1, -1);
                    $log->get_data("e_mail -> " . $e_mail);
                    if(empty($e_mail)) $e_mail = $headers['From'];
                    print_r($e_mail);
                    if($e_mail)
                    {
                        $in = change_charset('Windows-1251', 'UTF-8', $pop3->getBody($i));
                        $log->get_data("in -> " . $in);
                        if($array = get_message_from_mail($in, $e_mail)){
                            $log->get_data($array);
                            $answer = send_sms_from_array($array);
                            $log->get_data($answer);
                            $log->set_data();
                        }
                    }
                    $pop3->deleteMsg($i);
                }
            }
            $pop3->disconnect();
        }
        return FALSE;
    }

    function send_xml_from_file($log, $dir, $file)
    {
        echo $dir.','. $file;
        $log->get_data('Открываем файл ' . $dir . 'in/' . $file);
        $xml =file_get_contents($dir . 'in/' . $file);
        if($xml !== FALSE && strpos($xml, '</request>') !== FALSE)
        {
            echo '1';
            $log->get_data('Удаляем файл ' . $dir . 'in/' . $file);
            unlink($dir . 'in/' . $file);
            if(!file_exists($dir . 'in/' . $file))
            {
                if(strpos($xml, 'message') !== FALSE)
                    $resp = send_from_xml('sms', $xml);
                elseif(strpos($xml, 'id_sms') !== FALSE)
                    $resp = send_from_xml('state', $xml);
                else
                    $resp = send_from_xml('balance', $xml);
                $log->get_data('Записываем ответ в файл ' . $dir . 'out/' . $file);
                if(!is_dir($dir . 'out'))
                    mkdir($dir . 'out', 0777);
                $file_w = fopen($dir . 'out/' . $file, 'a');
                if($file !== FALSE)
                {
                    fwrite($file_w, $resp);
                    fclose($file_w);
                }
            }
        }
    }

    function set_url_wap_push($url)
    {
        $num_1 = strpos($url, '//');
        if($num_1 === FALSE)
            $return_url = chr(0x0B);
        else
        {
            $href = substr($url, 0, $num_1);
            if($href == 'http:' || $href == 'https:')
            {
                $www = substr($url, $num_1+2, 4);
                if($href == 'http:' && $www == 'www.')
                {
                    $return_url = chr(0x0D);
                    $url = substr($url, 11);
                }
                elseif($href == 'http:' && $www != 'www.')
                {
                    $return_url = chr(0x0C);
                    $url = substr($url, 7);
                }
                elseif($www == 'www.')
                {
                    $return_url = chr(0x0F);
                    $url = substr($url, 12);
                }
                else
                {
                    $return_url = chr(0x0E);
                    $url = substr($url, 8);
                }
            }
            else
                $return_url = chr(0x0B);
        }

        $url = str_replace('.com/', convert_hex('008503'), $url);
        $url = str_replace('.edu/', convert_hex('008603'), $url);
        $url = str_replace('.net/', convert_hex('008703'), $url);
        $url = str_replace('.org/', convert_hex('008803'), $url);

        $return_url .= chr(0x03) . $url . chr(0x00);
        return $return_url;
    }

    // $_message - строка состоящая из HEX байт
    // Возвращает строку символов соответствующих HEX байтам
    function convert_hex($_message)
    {
        $len = strlen($_message);
        for($i=0; $i<$len; $i+=2)
        {
            $tmp = hexdec(substr($_message, $i, 2));
            $return .= chr($tmp);
        }
        return $return;
    }

    // Получает код символа в позиции $from в строке $message
    function strdec($message, $from)
    {
        return ord(substr($message, $from, 1));
    }

    // $message - короткое сообщение, содержащее UDH
    // Возвращает UDH в виде массива
    function get_udh($message)
    {
        $udh = array();
        $length = strdec($message, 0)+1;
        if($length > strlen($message))
            return FALSE;
        for($i=1; $i<$length; $i++)
        {
            $iei_type = strdec($message, $i);
            $iei_len = strdec($message, ++$i);
            for($j=0; $j<$iei_len; $j++)
            {
                $udh[$iei_type][$j] = strdec($message, ++$i);
            }
        }
        $udh['message'] = substr($message, $length);
        return $udh;
    }

    function get_wap_push($short_message)
    {
        $num_1 = strpos($short_message, chr(0xC6));
        if($num_1 === FALSE)
            return FALSE;
        $num_1++;
        $num_2 = strpos($short_message, chr(0x01), $num_1);
        if($num_2 === FALSE)
            return FALSE;
        $all_url = substr($short_message, $num_1, $num_2-$num_1);
        $href = strdec($all_url, 0);
        switch($href)
        {
            case 0x0B:
            {
                $url = '';
                break;
            }
            case 0x0C:
            {
                $url = 'http://';
                break;
            }
            case 0x0D:
            {
                $url = 'http://www.';
                break;
            }
            case 0x0E:
            {
                $url = 'https://';
                break;
            }
            case 0x0F:
            {
                $url = 'https://www.';
                break;
            }
            default:
                return FALSE;
        }

        $num_3 = strpos($all_url, chr(0x03));
        if($num_3 === FALSE)
            return FALSE;
        $num_3++;
        $num_4 = strpos($all_url, chr(0x00), $num_3);
        $url .= substr($all_url, $num_3, $num_4-$num_3);
        $num_5 = strpos($all_url, chr(0x03), $num_4);
        if($num_5 !== FALSE)
        {
            $num_5++;
            $domain_zone = strdec($all_url, $num_4+1);
            switch($domain_zone)
            {
                case 0x85:
                {
                    $url .= '.com/';
                    break;
                }
                case 0x86:
                {
                    $url .= '.edu/';
                    break;
                }
                case 0x87:
                {
                    $url .= '.net/';
                    break;
                }
                case 0x88:
                {
                    $url .= '.org/';
                    break;
                }
                default:
                {
                    return FALSE;
                }
            }
            $num_6 = strpos($all_url, chr(0x00), $num_5);
            if($num_6 === FALSE)
                return FALSE;
            $url .= substr($all_url, $num_5, $num_6-$num_5);
        }
        $num_7 = strpos($short_message, chr(0x03), $num_2+1);
        if($num_7 === FALSE)
            return FALSE;
        $num_7++;
        $num_8 = strpos($short_message, chr(0x00), $num_7);
        $message = substr($short_message, $num_7, $num_8-$num_7);

        $return = array();
        $return['url'] = $url;
        $return['message'] = $message;
        return $return;
    }

    function set_param_sms($param)
    {
        if(is_array($param))
        {
            $message = '';
            foreach($param as $text)
            {
                $text = str_replace(chr(1), chr(1).chr(1), $text);
                $text = str_replace(chr(0), chr(1).chr(0), $text);
                $message .= $text . chr(0);
            }
            return $message;
        }
        else
         return FALSE;
    }

    function get_param_sms($text)
    {
        $result = array();
        $len = strlen($text);
        $tmp = '';
        for($i=0; $i<$len; $i++)
        {
            if($text[$i] == chr(1) && ($text[$i+1] == chr(1) || $text[$i+1] == chr(0)))
            {
                $tmp .= $text[$i+1];
                $i++;
            }
            elseif($text[$i] == chr(0))
            {
                $result[] = $tmp;
                $tmp = '';
            }
            else
                $tmp .= $text[$i];
        }
        $result[] = $tmp;
        return $result;
    }

    function get_vcard($text)
    {
        $tmp = get_param_sms($text);
        $count = count($tmp);
        $n = "\r\n";
        for($i=0; $i<$count; $i++)
        {
            if(($i<8 || $i>13) && $tmp[$i])
            {
                if(get_char_message($tmp[$i]) == 0x8)
                    $tmp[$i] = ';CHARSET=UTF-8:' . $tmp[$i] . $n;
                else
                    $tmp[$i] = ':' . $tmp[$i] . $n;
            }
        }

        $text = 'BEGIN:VCARD' . $n;
        $text .= 'VERSION:2.1' . $n;
        if($tmp[0])
            $text .= 'N' . $tmp[0];
        if($tmp[1])
            $text .= 'TEL;CELL' . $tmp[1];
        if($tmp[2])
            $text .= 'TEL;WORK' . $tmp[2];
        if($tmp[3])
            $text .= 'TEL;FAX' . $tmp[3];
        if($tmp[4])
            $text .= 'EMAIL' . $tmp[4];
        if($tmp[5])
            $text .= 'TITLE' . $tmp[5];
        if($tmp[6])
            $text .= 'ORG' . $tmp[6];
        if($tmp[7])
            $text .= 'URL' . $tmp[7];
        if($tmp[8] || $tmp[9] || $tmp[10] || $tmp[11] || $tmp[12] || $tmp[13])
        {
            $address = $tmp[8] . ';;' . $tmp[9] . ';' . $tmp[10] . ';' . $tmp[11] . ';' . $tmp[12] . ';' . $tmp[13];
            if(get_char_message($address) == 0x8)
                $text .= 'ADR;CHARSET=UTF-8:' . $address . $n;
            else
                $text .= 'ADR:' . $address . $n;
        }
        if($tmp[14])
            $text .= 'NOTE' . $tmp[14];
        $text .= 'END:VCARD';
        return $text;
    }

    function set_vcard($text)
    {
        $text = str_replace("\r", '', $text);
        $n = "\n";
        $tmp = explode($n, $text);
        $count_tmp = count($tmp);
        for($i=0; $i<$count_tmp; $i++)
        {
            unset($tmpp);
            $tmpp[$i] = explode(':', $tmp[$i], 2);
            $tmpp[$i][0] = explode(';', $tmpp[$i][0]);
            $name = mb_strtoupper($tmpp[$i][0][0]);
            unset($tmpp[$i][0][0]);
            if($name && $tmpp[$i][1])
            {
                if(is_array($tmpp[$i][0]))
                {
                    foreach($tmpp[$i][0] as $key => $val)
                    {
                        $return[$i][$name]['param'][mb_strtoupper($val)] = 1;
                    }
                }
                $return[$i][$name]['value'] = $tmpp[$i][1];
            }
        }

        $result = FALSE;
        if(is_array($return))
        {
            foreach($return as $teg)
            {
                if(isset($teg['N']))
                    $result['name'] = $teg['N']['value'];
                elseif(isset($teg['TEL']) && isset($teg['TEL']['param']['CELL']))
                    $result['phone_cell'] = $teg['TEL']['value'];
                elseif(isset($teg['TEL']) && isset($teg['TEL']['param']['WORK']))
                    $result['phone_work'] = $teg['TEL']['value'];
                elseif(isset($teg['TEL']) && isset($teg['TEL']['param']['FAX']))
                    $result['phone_fax'] = $teg['TEL']['value'];
                elseif(isset($teg['EMAIL']))
                    $result['e_mail'] = $teg['EMAIL']['value'];
                elseif(isset($teg['TITLE']))
                    $result['position'] = $teg['TITLE']['value'];
                elseif(isset($teg['ORG']))
                    $result['organization'] = $teg['ORG']['value'];
                elseif(isset($teg['URL']))
                    $result['url'] = $teg['URL']['value'];
                elseif(isset($teg['NOTE']))
                    $result['additional'] = $teg['NOTE']['value'];
                elseif(isset($teg['ADR']))
                {
                    $address = explode(';', $teg['ADR']['value'], 7);
                    if($address[0])
                        $result['address_post_office_box'] = $address[0];
                    if($address[2])
                        $result['address_street'] = $address[2];
                    if($address[3])
                        $result['address_city'] = $address[3];
                    if($address[4])
                        $result['address_region'] = $address[4];
                    if($address[5])
                        $result['address_postal_code'] = $address[5];
                    if($address[6])
                        $result['address_country'] = $address[6];
                }
            }
        }
        return $result;
    }

    function check_smpp_sender($smpp)
    {
        global $SQL;
         $query = 'select sql_cache num_send, reconnect, state_time, time_zone, bind, enquirelink_timeout, pause_if_error  from sms_aggregating where type="SMPP" and ip!="" and port!="" and num_send>0 and id_aggregating="' . $smpp->id_aggregating . '"';
         $result = $SQL->query($query);
         if($SQL->num_rows($result))
         {
             $smpp->state_time = $SQL->result($result, 0, 'state_time');
             $smpp->time_zone = $SQL->result($result, 0, 'time_zone');
             $check_smpp['num_send'] = $SQL->result($result, 0, 'num_send');
             $check_smpp['pause_if_error'] = $SQL->result($result, 0, 'pause_if_error');
             $check_smpp['enquirelink_timeout'] = $SQL->result($result, 0, 'enquirelink_timeout');
            $reconnect = $SQL->result($result, 0, 'reconnect');
            if($reconnect == '2')
            {
                $smpp->type = $SQL->result($result, 0, 'bind');
                $query = 'update sms_aggregating set reconnect="1" where id_aggregating="' . $smpp->id_aggregating . '"';
                $SQL->query($query);
                $smpp->_socket = FALSE;
                $check_smpp['start_smpp'] = $smpp->StartSmpp();
            }
            return $check_smpp;
        }
        else
        {
            $smpp->End();
            $smpp->set_data();
            exit();
        }
    }

    function check_base($id_user, $id_base)
    {
        global $SQL;
        $query = 'select name_base from sms_base where id_base="' . $id_base . '" and id_user="' . $id_user . '"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
        {
            $name_base = $SQL->result($result, 0, 'name_base');
            return $name_base;
        }
        else
            return FALSE;
    }

    function check_id_base($id_user, $id_phone)
    {
        global $SQL;
        $query = 'select sms_base.id_base from sms_base, sms_phone where sms_base.id_base=sms_phone.id_base and id_user="' . $id_user . '" and id_phone="' . $id_phone . '"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
        {
            $id_base = $SQL->result($result, 0, 'sms_base.id_base');
            return $id_base;
        }
        else
            return FALSE;
    }

    function get_price_admin($price)
    {
        $price = str_replace(',', '.', $price);
        $price = str_replace(' ', '', $price);
        return $price;
    }

    function check_payment($id_client, $id_partner, $type, $id_payment)
    {
        global $SQL;
        if($id_client > 0)
            $query = 'select id_package_user, id_user from sms_package_user where type_payment="' . $type . '" and id_payment="' . $id_payment . '" limit 0, 1';
        else
            $query = 'select id_package_partner, id_partner from sms_package_partner where type_payment="' . $type . '" and id_payment="' . $id_payment . '" limit 0, 1';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
        {
            if($id_client > 0)
            {
                for($i=0; $i<$num; $i++)
                {
                    $id_user = $SQL->result($result, $i, 'id_user');
                    $param = get_param_id_user($id_user);
                    $id_partner_ = $param['id_partner'];
                    if($id_partner_ == $id_partner)
                        return TRUE;
                }
                return FALSE;
            }
            else
            {
                $id_partner_ = $SQL->result($result, 0, 'id_partner');
                if($id_partner_ == $id_partner)
                    return TRUE;
                else
                    return FALSE;
            }
        }
        else
            return FALSE;
    }

    function get_one_price_sms($id_client, $id_partner, $id_currency, $MCC, $MNC, $limit_price, $id_group_price)
    {
        $prices = get_price_sms($id_client, $id_partner, $id_currency, 'sms', $limit_price);
        foreach($prices as $price)
        {
            if($price['MCC'] == $MCC && $price['MNC'] == $MNC && $price['id_group_price'] == $id_group_price)
            {
                return $price['price_sms'];
            }
        }
        return FALSE;
    }

    function get_group_price($id_client, $id_partner=FALSE)
    {
        global $SQL;
        $b_id_partner=$id_partner===FALSE;
        if($b_id_partner)
        {
            $user = get_param_id_user($id_client);
            $id_partner = $user['id_partner'];
        }

        $query = 'select sql_cache id_group_price, id_client, id_partner from sms_client_prices where (id_partner="' . $id_partner . '" or id_partner=0) and (id_client="' . $id_client . '" or id_client=0) order by id_partner desc, if(id_client=0, 2, 1) asc';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
        {
            for($i=0; $i<$num; $i++)
            {
                $return[$i]['id_group_price'] = $SQL->result($result, $i, 'id_group_price');
                $return[$i]['id_client'] = $SQL->result($result, $i, 'id_client');
                $return[$i]['id_partner'] = $SQL->result($result, $i, 'id_partner');
            }
            return $return;
        }
        else
            return FALSE;
    }

    function get_price_sms($id_client, $id_partner=FALSE, $id_currency=FALSE, $type_sms=FALSE, $sum = FALSE)
    {
        global $SQL;
        $b_id_partner=$id_partner===FALSE;
        $b_id_currency=$id_currency===FALSE;
        if($b_id_partner || $b_id_currency)
        {
            if($id_client>0)
            {
                $user = get_param_id_user($id_client);
                if($b_id_partner)
                    $id_partner = $user['id_partner'];
                if($b_id_currency)
                    $id_currency = $user['id_currency'];
            }

        }
        if(is_array($id_group_prices = get_group_price($id_client, $id_partner)))
        {
            $j = 0;
            foreach($id_group_prices as $id_group_price)
            {
                if(!isset($return))
                {
                    $id_client_last = FALSE;
                    $id_partner_last = FALSE;
                }
                if($sum)
                    $sum_where = ' and num_sms_from*price_sms<="' . addslashes($sum) . '"';
                else
                    $sum_where = '';
                if($type_sms)
                   $where_type_sms=' type_sms="'.$type_sms.'" and ';
                else
                   $where_type_sms=' ';
                $query = 'select sql_cache sp1.id_price, sp1.MCC, sp1.MNC, sp1.price_sms, sp1.type_sms, sp1.deliver_payment from sms_prices as sp1,
                (select max(num_sms_from) as num_sms_from, MCC,MNC,type_sms from sms_prices where '.$where_type_sms.' id_currency="' . $id_currency . '"' . $sum_where . ' and id_group_price="' . $id_group_price['id_group_price'] . '" group by MCC,MNC,type_sms) as sp2 where  sp1.id_currency="' . $id_currency . '" and sp1.id_group_price="' . $id_group_price['id_group_price'] . '" and sp1.type_sms=sp2.type_sms and sp1.MCC=sp2.MCC and sp1.MNC=sp2.MNC and sp1.num_sms_from=sp2.num_sms_from';
                //         sp1.type_sms="'.$type_sms.'" and
                $result = $SQL->query($query);
                $num = $SQL->num_rows($result);
                if($num)
                {
                    if($id_client_last === FALSE && $id_partner_last === FALSE)
                    {
                        $id_client_last = $id_group_price['id_client'];
                        $id_partner_last = $id_group_price['id_partner'];
                    }
                    if($id_client_last != $id_group_price['id_client'] || $id_partner_last != $id_group_price['id_partner'])
                        return $return;

                    for($i=0; $i<$num; $i++, $j++)
                    {
                        $return[$j]['id_price'] = $SQL->result($result, $i, 'sp1.id_price');
                        $return[$j]['MCC'] = $SQL->result($result, $i, 'sp1.MCC');
                        $return[$j]['MNC'] = $SQL->result($result, $i, 'sp1.MNC');
                        $return[$j]['price_sms'] = $SQL->result($result, $i, 'sp1.price_sms');
                        $return[$j]['type_sms'] = $SQL->result($result, $i, 'sp1.type_sms');
                        $return[$j]['deliver_payment'] = $SQL->result($result, $i, 'sp1.deliver_payment');
                        $return[$j]['id_group_price'] = $id_group_price['id_group_price'];
                    }
                }
            }
        }

        if(is_array($return))
            return $return;
        else
            return FALSE;
    }

    function get_num_sms($id_client, $id_partner, $id_currency, $sms = FALSE)
    {
        global $SQL;
        if(is_array($id_group_prices = get_group_price($id_client, $id_partner)))
        {
            $id_client_last = FALSE;
            $id_partner_last = FALSE;
            $j = 0;
            foreach($id_group_prices as $id_group_price)
            {
                $query = 'select sql_cache sp1.id_price, sp1.MCC, sp1.MNC, sp1.price_sms,sp1.type_sms from sms_prices as sp1,
                (select max(num_sms_from) as num_sms_from,  MCC,MNC , type_sms  from sms_prices where id_currency="' . $id_currency . '" and num_sms_from<="' . addslashes($sms) . '" and id_group_price="' . $id_group_price['id_group_price'] . '" group by MCC,MNC,type_sms) as sp2 where sp1.id_currency="' . $id_currency . '" and sp1.id_group_price="' . $id_group_price['id_group_price'] . '" and sp1.type_sms=sp2.type_sms and sp1.MCC=sp2.MCC and sp1.MNC=sp2.MNC and sp1.num_sms_from=sp2.num_sms_from';
                $result = $SQL->query($query);
                $num = $SQL->num_rows($result);
                if($num)
                {
                    if($id_client_last === FALSE && $id_partner_last === FALSE)
                    {
                        $id_client_last = $id_group_price['id_client'];
                        $id_partner_last = $id_group_price['id_partner'];
                    }
                    if($id_client_last != $id_group_price['id_client'] || $id_partner_last != $id_group_price['id_partner'])
                        return $return;
                    for($i=0; $i<$num; $i++, $j++)
                    {
                        $return[$j]['id_price'] = $SQL->result($result, $i, 'id_price');
                        $return[$j]['MCC'] = $SQL->result($result, $i, 'sp1.MCC');
                        $return[$j]['MNC'] = $SQL->result($result, $i, 'sp1.MNC');
                        $return[$j]['price_sms'] = $SQL->result($result, $i, 'price_sms');
                        $return[$j]['id_group_price'] = $id_group_price['id_group_price'];
                        $return[$j]['type_sms'] = $SQL->result($result, $i, 'sp1.type_sms');
                    }
                }
            }
        }
        if(is_array($return))
            return $return;
        else
            return FALSE;
    }

    function add_package($id_client, $id_partner, $id_currency, $sum, $type_payment, $id_payment, $payment_info, $credit=0, $pay='1')
    {
        global $SQL, $global_languages;
        $prices = get_price_sms($id_client, $id_partner, $id_currency, FALSE, $sum);
        if(is_array($prices))
        {
            $date_start = date('Y-m-d');
            $date_stop = date_add_m_d($date_start, 120);
            if($credit)
               $set_credit=', credit="'.$credit.'" ';
            if($id_client > 0)
            {
                if($type_payment == 'test_sms')
                    $test_sms = ', test_sms="1"';
                else
                    $test_sms = '';
                $query = 'insert into sms_package_user set id_user="' . $id_client . '", limit_price="' . $sum . '", pay="' . $pay . '", date_start="' . $date_start . '", date_stop="' . $date_stop . '", id_currency="' . $id_currency . '", type_payment="' . $type_payment . '", id_payment="' . $id_payment . '", payment_info="' . $payment_info . '"' . $test_sms. $set_credit;
                $SQL->query($query);
                $id_package = $SQL->insert_id();
                $query = 'select count(*) as num from sms_wait where id_user="' . $id_client . '"';
                $result = $SQL->query($query);
                $num = $SQL->result($result, 0, 'num');
                if($num)
                    set_message($id_client, 'У вас есть неоплаченные SMS. Для их отправки или удаления перейдите в раздел <a href="javascript:open_href(\'/\'+current_language+\'/cabinet/wait.html\');" class="message_link_1">"SMS ожидающие оплаты"</a>.');
            }
            else
            {
                $query = 'insert into sms_package_partner set id_partner="' . (-1*$id_client) . '", limit_price="' . $sum . '", date_start="' . $date_start . '", date_stop="' . $date_stop . '", id_currency="' . $id_currency . '", type_payment="' . $type_payment . '", id_payment="' . $id_payment . '"';
                $SQL->query($query);
                $id_package = $SQL->insert_id();
            }
            foreach($prices as $prise)
            {
                if($id_client > 0)
                    $query = 'insert into sms_price_user set deliver_payment="' . $prise['deliver_payment'] . '", type_sms="' . $prise['type_sms'] . '", price_sms="' . $prise['price_sms'] . '", id_package_user="' . $id_package . '", MCC="' . $prise['MCC'] . '", MNC="' . $prise['MNC'] . '", id_group_price="' . $prise['id_group_price'] . '"';
                else
                    $query = 'insert into sms_price_partner set deliver_payment="' . $prise['deliver_payment'] . '", type_sms="' . $prise['type_sms'] . '", price_sms="' . $prise['price_sms'] . '", id_package_partner="' . $id_package . '", MCC="' . $prise['MCC'] . '", MNC="' . $prise['MNC'] . '", id_group_price="' . $prise['id_group_price'] . '"';
                $SQL->query($query);
            }
            if($type_payment != 'test_sms')
            {
                if($id_client > 0){
                    $query = 'update sms_send_users set send_sms_small_balance="1" where id_user="' . $id_client . '"';
                    send_partner_mail($id_client, "2024", $id_partner, array("MONEY"=>$sum));
                } else
                    $query = 'update sms_partners set send_sms_small_balance="1" where id_partner="' . (-1*$id_client) . '"';
                $SQL->query($query);
            }

            return $id_package;
        }
        else
            return FALSE;
    }

    function check_partner_user($id_user, $where_partner)
    {
        global $SQL;
        $query = 'select count(*) as num from sms_send_users where id_user="' . $id_user . '"' . $where_partner;
        $result = $SQL->query($query);
        if(!$SQL->result($result, 0, 'num'))
            exit();
    }

    function ip_in_range($ip, $range) {
            if (strpos($range, '/') !== false) {
                // $range is in IP/NETMASK format
                list($range, $netmask) = explode('/', $range, 2);
                if (strpos($netmask, '.') !== false) {
                    // $netmask is a 255.255.0.0 format
                    $netmask = str_replace('*', '0', $netmask);
                    $netmask_dec = ip2long($netmask);
                    return ( (ip2long($ip) & $netmask_dec) == (ip2long($range) & $netmask_dec) );
                } else {
                    // $netmask is a CIDR size block
                    // fix the range argument
                    $x = explode('.', $range);
                    while(count($x)<4) $x[] = '0';
                    list($a,$b,$c,$d) = $x;
                    $range = sprintf("%u.%u.%u.%u", empty($a)?'0':$a, empty($b)?'0':$b,empty($c)?'0':$c,empty($d)?'0':$d);
                    $range_dec = ip2long($range);
                    $ip_dec = ip2long($ip);
                    
                    # Strategy 1 - Create the netmask with 'netmask' 1s and then fill it to 32 with 0s
                    #$netmask_dec = bindec(str_pad('', $netmask, '1') . str_pad('', 32-$netmask, '0'));

                    # Strategy 2 - Use math to create it
                    $wildcard_dec = pow(2, (32-$netmask)) - 1;
                    $netmask_dec = ~ $wildcard_dec;
                    return (($ip_dec & $netmask_dec) == ($range_dec & $netmask_dec));
                }
            } else {
                // range might be 255.255.. or 1.2.3.0-1.2.3.255
                if (strpos($range, '*') !==false) { // a.b.. format
                    // Just convert to A-B format by setting * to 0 for A and 255 for B
                    $lower = str_replace('*', '0', $range);
                    $upper = str_replace('*', '255', $range);
                    $range = "$lower-$upper";
                }

                if (strpos($range, '-')!==false) { // A-B format
                    list($lower, $upper) = explode('-', $range, 2);
                    $lower_dec = (float)sprintf("%u",ip2long($lower));
                    $upper_dec = (float)sprintf("%u",ip2long($upper));
                    $ip_dec = (float)sprintf("%u",ip2long($ip));
                    return ( ($ip_dec>=$lower_dec) && ($ip_dec<=$upper_dec) );
                }

                //echo 'Range argument is not in 1.2.3.4/24 or 1.2.3.4/255.255.255.0 format';
                return false;
            }
        }
    function check_user_ip($id_user, $ip)
    {
        global $SQL;
        $query = 'select * from sms_ip_smpp where ip="' . $ip . '" and id_user="' . $id_user . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
            return TRUE;
        else
        {
            $query = 'select * from sms_ip_smpp where id_user="' . $id_user . '" and (ip LIKE ("%/%") or ip LIKE ("%*%") or ip LIKE ("%-%"))';
            $result = $SQL->query($query);
            $num = $SQL->num_rows($result);
            if($num>0){
                for($i=0; $i<$num; $i++)
                {
                    $range = $SQL->result($result, $i, 'ip');
                    if(ip_in_range($ip, $range))
                        return TRUE;
                        
                }
                return FALSE;
            }
            else{
                return FALSE;
            }
        }
    }

    function get_id_partner($login)
    {
        global $SQL;
        $query = 'select sql_cache id_partner from sms_partners where login="' . $login . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
            return $SQL->result($result, 0, 'id_partner');
        else
            return FALSE;
    }

    function get_id_currency($short_title)
    {
        global $SQL;
        $query = 'select sql_cache id_currency from sms_currency where short_title="' . $short_title . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
            return $SQL->result($result, 0, 'id_currency');
        else
            return FALSE;
    }

    function get_param_user($login)
    {
        global $SQL;
        $query = 'select sql_cache id_user, id_partner, id_currency, id_manager from sms_send_users where login="' . $login . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            $id_partner = $SQL->result($result, 0, 'id_partner');
            $id_currency = $SQL->result($result, 0, 'id_currency');
            $id_manager = $SQL->result($result, 0, 'id_manager');
            return array('id_user' => $id_user, 'id_partner' => $id_partner, 'id_currency' => $id_currency, 'id_manager' => $id_manager);
        }
        else
            return FALSE;
    }
    function user($id_user=FALSE,$param=FALSE)
    {
        global $GLOBAL_CACHE_DATA;
        if($id_user)
        {
            if(isset($GLOBAL_CACHE_DATA['get_param_id_user'][$id_user]))
                get_param_id_user($id_user);
            if($param)
                return $GLOBAL_CACHE_DATA['get_param_id_user'][$id_user][$param];
            return $GLOBAL_CACHE_DATA['get_param_id_user'][$id_user];
        }
    }
    function get_param_id_user($id_user)
    {
        global $SQL, $GLOBAL_CACHE_DATA;
        if(isset($GLOBAL_CACHE_DATA['get_param_id_user'][$id_user]))
            return $GLOBAL_CACHE_DATA['get_param_id_user'][$id_user];
        else
        {
            $query = 'select sql_cache id_currency, id_partner, id_manager, url_get_sms, email_get_sms, send_incoming_smpp, login, name, all_originator, control_text, local_time, phone, send_sms, pay_control from sms_send_users where id_user="' . $id_user . '"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
            {
                $id_currency = $SQL->result($result, 0, 'id_currency');
                $id_partner = $SQL->result($result, 0, 'id_partner');
                $id_manager = $SQL->result($result, 0, 'id_manager');
                $url_get_sms = $SQL->result($result, 0, 'url_get_sms');
                $email_get_sms = $SQL->result($result, 0, 'email_get_sms');
                $send_incoming_smpp = $SQL->result($result, 0, 'send_incoming_smpp');
                $login = $SQL->result($result, 0, 'login');
                $name = $SQL->result($result, 0, 'name');
                $all_originator = $SQL->result($result, 0, 'all_originator');
                $control_text = $SQL->result($result, 0, 'control_text');
                $local_time = $SQL->result($result, 0, 'local_time');
                $send_sms = $SQL->result($result, 0, 'send_sms');
                $phone = $SQL->result($result, 0, 'phone');
                $pay_control = $SQL->result($result, 0, 'pay_control');// Снимать денежки с неодобренных смс
                $GLOBAL_CACHE_DATA['get_param_id_user'][$id_user] = array('id_currency' => $id_currency, 'id_partner' => $id_partner, 'id_manager' => $id_manager, 'url_get_sms' => $url_get_sms, 'email_get_sms' => $email_get_sms, 'send_incoming_smpp' => $send_incoming_smpp, 'login' => $login, 'name' =>$name, 'all_originator' => $all_originator, 'control_text' => $control_text, 'local_time' => $local_time, 'phone'=>$phone,'send_sms'=>$send_sms,'pay_control'=>$pay_control);
                return $GLOBAL_CACHE_DATA['get_param_id_user'][$id_user];
            }
            else
                return FALSE;
        }
    }

    function get_param_id_help($id_help_title)
    {
        global $SQL;
        $query = 'select sql_cache e_mail, id_partner, title from help_title where id_help_title="' . $id_help_title . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $e_mail = $SQL->result($result, 0, 'e_mail');
            $id_partner = $SQL->result($result, 0, 'id_partner');
            $title = $SQL->result($result, 0, 'title');
            return array('e_mail' => $e_mail, 'id_partner' => $id_partner, 'title' => $title);
        }
        else
            return FALSE;
    }

    function get_param_id_partner($id_partner)
    {
        global $SQL;
        $query = 'select sql_cache id_currency from sms_partners where id_partner="' . $id_partner . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_currency = $SQL->result($result, 0, 'id_currency');
            return array('id_currency' => $id_currency);
        }
        else
            return FALSE;
    }

    function get_param_site($id_partner,$id_user=FALSE)
    {
        global $SQL;
        $return = array();
        $query_adjustment = 'select sa.adjustment, sav.value from sms_adjustment as sa left join sms_adjustment_value sav on(sa.id_adjustment=sav.id_adjustment) where (sa.type="partner" and (sav.id_partner="' . $id_partner . '" or sav.id_partner is NULL)) or (sa.type="main" and (sav.id_partner=0 or sav.id_partner is NULL)) order by sa.priority asc';
        $result_adjustment = $SQL->query($query_adjustment);
        $num_adjustment = $SQL->num_rows($result_adjustment);
        for($j=0; $j<$num_adjustment; $j++)
        {
            $adjustment = $SQL->result($result_adjustment, $j, 'sa.adjustment');
            $return[$adjustment] = $SQL->result($result_adjustment, $j, 'sav.value');
        }
        if($return['SIGNATURE_LEADER']){
            if(!@fopen($return['SIGNATURE_LEADER'], 'r')){
                $return['SIGNATURE_LEADER'] = '/pay/signature/leader/'.$id_partner.'.'.$return['SIGNATURE_LEADER'];
            }
        } else { $return['SIGNATURE_LEADER'] = '/pay/signature/leader/'.$id_partner.'.'.$return['SIGNATURE_LEADER']; }
        if($return['STAMP']){
            if(!@fopen($return['STAMP'], 'r')){
                $return['STAMP'] = '/pay/stamp/'.$id_partner.'.'.$return['STAMP'];
            }
        } else { $return['STAMP'] = '/pay/stamp/'.$id_partner.'.'.$return['STAMP']; }
        if($return['SIGNATURE_LEADER']){
            if(!@fopen($return['SIGNATURE_CHIEF'], 'r')){
                $return['SIGNATURE_CHIEF'] = '/pay/signature/chief/'.$id_partner.'.'.$return['SIGNATURE_CHIEF'];
            }
        } else { $return['SIGNATURE_CHIEF'] = '/pay/signature/chief/'.$id_partner.'.'.$return['SIGNATURE_CHIEF']; }
        //$return['SIGNATURE_LEADER'] = 'signature/leader/'.$id_partner.'.'.$return['SIGNATURE_LEADER'];
        //$return['STAMP'] = 'stamp/'.$id_partner.'.'.$return['STAMP'];
        //$return['SIGNATURE_CHIEF'] = 'signature/chief/'.$id_partner.'.'.$return['SIGNATURE_CHIEF'];

        if($id_user)
        {
            $query = 'SELECT * FROM sms_companys where id_company in (SELECT id_company FROM sms_send_users where id_user="'.$id_user.'") and id_partner ="'.$id_partner.'"';
            $result = $SQL->query($query);
            if($result)
            {
                if($res = $SQL->fetch_assoc($result))
                {
                    $res['NAME_PAYMENT'] = $res['name'];
                    unset($res['name']);
                    foreach($res as $key=>$v)
                    {
                        if(strpos($key,'file_')===0)
                        {
                            $key=substr($key,5);
                            $v = pay_file_get($v);
                            if($v)
                                $return[strtoupper($key)]='/pay/'.$v;
                        }else
                            $return[strtoupper($key)]=$v;
                    }
                }
            }
        }
        return $return;
    }

    function check_e_mail($e_mail)
    {
        if(preg_match('/^[\.A-Za-z0-9_\-]+@[\.A-Za-z0-9_-]+[\.][A-Za-z]+$/', $e_mail))
            return TRUE;
        else
            return FALSE;
    }

    function check_ip($ip)
    {
        if(ip2long($ip))
            return TRUE;
        else{
            if(strpos($ip, '/') !== false || strpos($ip, '*') !== false || strpos($ip, '-') !== false){
            return TRUE;
            }
        return FALSE;
        }   
        
    }

    function get_user_incoming($source_addr, $phone, $short_message, $id_aggregating = '', $id = '')
    {
        global $SQL;
        $source_addr_orig = $source_addr;
        $phone_orig = $phone;
        $short_message_orig = $short_message;
        $find = strpos($source_addr,'#');
        if($find!==FALSE)
            $source_addr = substr($source_addr,0,$find);
        $short_message = addslashes(mb_strtoupper($short_message));
        $query = 'select sql_cache id_phone, id_user, prefix from sms_receive_sms where phone="' . addslashes($phone) . '" and (prefix=SUBSTRING("' . $short_message . '", 1, CHAR_LENGTH(prefix)) or prefix="") order by prefix desc';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_phone = $SQL->result($result, 0, 'id_phone');
            $id_user = $SQL->result($result, 0, 'id_user');
            $prefix = $SQL->result($result, 0, 'prefix');
            $param = get_param_id_user($id_user);

            if($id_aggregating && $id)
            {
                $query = 'select id_sms from sms_incoming where id_aggregating="' . $id_aggregating . '" and id_operator="' . addslashes($id) . '"';
                $result = $SQL->query($query);
                $num = $SQL->num_rows($result);
            }
            else
                $num = 0;

            if($num == 0)
            {
                $query = 'insert into sms_incoming set id_user="' . $id_user . '", id_partner="' . $param['id_partner'] . '", prefix="' . addslashes($prefix) . '", originator="' . addslashes($source_addr) . '", text_sms="' . addslashes($short_message) . '", phone="' . addslashes($phone) . '", id_aggregating="' . $id_aggregating . '", id_operator="' . addslashes($id) . '"';
                $SQL->query($query);
                if($param['url_get_sms'])
                {
                    $query = 'insert into sms_incoming_get set id_user="' . $id_user . '", prefix="' . addslashes($prefix) . '", originator="' . addslashes($source_addr) . '", text_sms="' . addslashes($short_message) . '", phone="' . addslashes($phone) . '"';
                    $SQL->query($query);
                }
                if($param['email_get_sms'])
                {
                    $query = 'insert into sms_incoming_email set id_user="' . $id_user . '", id_partner="' . $param['id_partner'] . '", prefix="' . addslashes($prefix) . '", originator="' . addslashes($source_addr) . '", text_sms="' . addslashes($short_message) . '", phone="' . addslashes($phone) . '"';
                    $SQL->query($query);
                }
                if($param['send_incoming_smpp'])
                {
                    $query = 'insert into sms_incoming_smpp set id_user="' . $id_user . '", originator="' . addslashes($source_addr) . '", text_sms="' . addslashes($short_message) . '", phone="' . addslashes($phone) . '"';
                    $SQL->query($query);
                }

                $query = 'select sql_cache originator, answer from sms_receive_sms_answer where id_phone="' . $id_phone . '" and "' . addslashes($short_message). '" like text_sms order by priority asc';
                $result = $SQL->query($query);
                if($SQL->num_rows($result))
                {
                    $originator = $SQL->result($result, 0, 'originator');
                    $answer = $SQL->result($result, 0, 'answer');
                    $send_sms1 = send_sms($id_user, $param['all_originator'], $param['control_text'], $param['id_partner'], $source_addr, $originator, 'sms', $answer, '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '0000-00-00', '00:00', 'Ответные СМС ' . $phone);
                }
                if($id_user=='3623'){
                    
                    $send_sms2 = send_sms($id_user, $param['all_originator'], $param['control_text'], $param['id_partner'], $source_addr, $phone_orig, 'sms', $short_message_orig, '', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '0000-00-00', '00:00', 'Ответные СМС МТС' . $phone_orig);
                }
                return array('id_user' => $id_user, 'id_partner' => $param['id_partner'], 'url_get_sms' => $param['url_get_sms'], 'email_get_sms' => $param['email_get_sms'], 'send_incoming_smpp' => $param['send_incoming_smpp'], 'prefix' => $prefix, 'id_phone' => $id_phone, 'status' => print_r($send_sms2,TRUE));
            }
            else
                return array('id_user' => 0, 'id_partner' => 0, 'id_phone' => 0);
        }
        else
            return array('id_user' => 0, 'id_partner' => 0, 'id_phone' => 0);
    }

    function check_group_price($id_group_price, $id_partner)
    {
        global $SQL;
        $query = 'select id_group_price from sms_group_prices where id_group_price="' . $id_group_price . '" and id_partner="' . $id_partner . '"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
            return TRUE;
        else
            return FALSE;
    }

    function check_price($id_price, $id_partner)
    {
        global $SQL;
        $query = 'select id_group_price from sms_prices where id_price="' . $id_price . '"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
        {
            $id_group_price = $SQL->result($result, 0, 'id_group_price');
            return check_group_price($id_group_price, $id_partner);
        }
        else
            return FALSE;
    }

    function update_packages($id_group_price, $MCC, $MNC, $id_currency, $type = 'user')
    {
        global $SQL;
        $query = 'select * from sms_package_' . $type . ' as spa, sms_price_' . $type . ' as spr where spa.id_package_' . $type . '=spr.id_package_' . $type . ' and spr.MCC="' . $MCC . '" and spr.MNC="' . $MNC . '" and spa.id_currency="' . $id_currency . '" and spr.id_group_price="' . $id_group_price . '"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        $count = 0;
        for($i=0; $i<$num; $i++)
        {
            $id_client = $SQL->result($result, $i, 'spa.id_' . $type);
            $id_package = $SQL->result($result, $i, 'spa.id_package_' . $type);
            $limit_price = $SQL->result($result, $i, 'spa.limit_price');
            $id_currency = $SQL->result($result, $i, 'spa.id_currency');

            if($type == 'user')
            {
                $param = get_param_id_user($id_client);
                $id_partner = $param['id_partner'];
                $id_client_ = $id_client;
            }
            else
            {
                $id_partner = 0;
                $id_client_ = -1*$id_client;
            }

            $prices = get_one_price_sms($id_client_, $id_partner, $id_currency, $MCC, $MNC, $limit_price, $id_group_price);
            if($prices)
            {
                $query = 'update sms_price_' . $type . ' set price_sms="' . $prices . '" where id_package_' . $type . '="' . $id_package . '" and MCC="' . $MCC . '" and  MNC="' . $MNC . '" and id_group_price="' . $id_group_price . '"';
                $SQL->query($query);
                $count++;
            }
        }
        return $count;
    }

    function get_type_file($name)
    {
        $num = strrpos($name, '.');
        if($num !== FALSE)
            $type = substr($name, $num+1);
        else
            $type = FALSE;
        return $type;
    }

    function get_small_balance($value)
    {
        $value_2 = $value;
        $value = get_phone($value);
        if(strpos($value_2, '%') !== FALSE)
            $value .= ' %';
        return $value;
    }

    function get_name_delivery($id_name_delivery)
    {
        global $SQL, $global_name_delivery;
        if(isset($global_name_delivery[$id_name_delivery]))
            return $global_name_delivery[$id_name_delivery];
        else
        {
            $query = 'select sql_cache name_delivery from sms_name_delivery where id_name_delivery="' . $id_name_delivery . '"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
            {
                $name_delivery = $SQL->result($result, 0, 'name_delivery');
                $global_name_delivery[$id_name_delivery] = $name_delivery;
                return $name_delivery;
            }
            else
                return FALSE;
        }
    }

    function get_param_id_smpp_error($id_smpp_error)
    {
        global $SQL, $global_smpp_error;
        if(isset($global_smpp_error[$id_smpp_error]))
            return $global_smpp_error[$id_smpp_error];
        else
        {
            $query = 'select sql_cache code, name from sms_smpp_error where id_smpp_error="' . $id_smpp_error . '"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
            {
                $code = $SQL->result($result, 0, 'code');
                $name = $SQL->result($result, 0, 'name');
            }
            else
            {
                $code = 0;
                $name = '';
            }
            $global_smpp_error[$id_smpp_error]['code_dec'] = $code;
            $code1 = $code%256;
            $tmp1 = ($code-$code1)/256;
            $code2 = $tmp1%256;
            $code3 = (($tmp1-$code2)/256)%256;
            $global_smpp_error[$id_smpp_error]['code_string'] = chr($code3) . chr($code2) . chr($code1);
            $global_smpp_error[$id_smpp_error]['name'] = $name;
            return $global_smpp_error[$id_smpp_error];
        }
    }

    function get_param_id_currency($id_currency)
    {
        global $SQL;
        $query = 'select sql_cache title, short_title, test_balance from sms_currency where id_currency="' . $id_currency . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $title = $SQL->result($result, 0, 'title');
            $short_title = $SQL->result($result, 0, 'short_title');
            $test_balance = $SQL->result($result, 0, 'test_balance');
            return array('title' => $title, 'short_title' => $short_title, 'test_balance' => $test_balance);
        }
        else
            return FALSE;
    }

    function get_param_phone_code($MCC,$MNC)
    {
        global $SQL;
        $query = 'select sql_cache * from sms_phone_code_title where MCC="' . $MCC . '" and  MNC="' . $MNC . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $title = $SQL->result($result, 0, 'title');
            $id_phone_code = $SQL->result($result, 0, 'id_phone_code');
            $seen_in_pay = $SQL->result($result, 0, 'seen_in_pay');
            return array('title' => $title,'id_phone_code' => $id_phone_code, 'seen_in_pay' => $seen_in_pay);
        }
        else
            return FALSE;
    }

    function get_param_id_phone_code($id_phone_code)
    {
        global $SQL;
        $query = 'select sql_cache * from sms_phone_code_title where id_phone_code="' . $id_phone_code . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
           $MCC = $SQL->result($result, 0, 'MCC');
           $MNC = $SQL->result($result, 0, 'MNC');
           $title = $SQL->result($result, 0, 'title');
           $seen_in_pay = $SQL->result($result, 0, 'seen_in_pay');
            return array('title' => $title, 'seen_in_pay' => $seen_in_pay,'MCC' => $MCC,'MNC' => $MNC);
        }
        else
            return FALSE;
    }

    function get_href_value($value)
    {
        if($value == 'all')
            $value = '';
        return $value;
    }

    function set_href_value($value)
    {
        if(!$value && $value!==0 && $value!=='0')
            $value = 'all';
        return $value;
    }

    function set_error($error)
    {
        global $error_page;
        if(count($error_page) < 100)
            $error_page[] = $error;
    }

    function slashes_control($text)
    {
        $text = str_replace('\\\/', '/', $text);
        $text = str_replace('/', '\\\/', $text);
        return $text;
    }


    // Разбирает текст отчета о доставке и возвращает массив значаний
    function get_param_state($message)
    {
        $return = array();
        $message = str_replace('submit date', 'submit_date', $message);
        $message = str_replace('done date', 'done_date', $message);
        if(strpos($message, 'done_date') !== FALSE)
        {
            $message = str_replace(chr(0x00), '', $message);
            $tmp_zero = explode(' ', $message, 8);
            $num_1 = strpos($tmp_zero[0], ':');
            if($num_1 === FALSE)
                $num_1 = 0;
            else
                $num_1++;
            $return['id_message'] = substr($tmp_zero[0], $num_1, strlen($tmp_zero[0])-$num_1);

            $num_1 = strpos($tmp_zero[3], ':');
            if($num_1 === FALSE)
                $num_1 = 0;
            else
                $num_1++;
            $return['submit_date'] = get_time_from_state(substr($tmp_zero[3], $num_1, 10));

            $num_2 = strpos($tmp_zero[4], ':');
            if($num_2 === FALSE)
                $num_2 = 0;
            else
                $num_2++;
            $return['done_date'] = get_time_from_state(substr($tmp_zero[4], $num_2, 10));

            $num_3 = strpos($tmp_zero[5], ':');
            if($num_3 === FALSE)
                $num_3 = 0;
            else
                $num_3++;
            $return['stat'] = substr($tmp_zero[5], $num_3, 7);

            $num_4 = strpos($tmp_zero[6], ':');
            if($num_4 === FALSE)
                $num_4 = 0;
            else
                $num_4++;
            $return['err'] = substr($tmp_zero[6], $num_4, 3);

            $num_1 = strpos($message, 'netwerr:');
            if($num_1 !== FALSE)
                $return['network_error'] = substr($message, $num_1+8);

        }
        else
        {
            $num_1 = strpos($message, chr(0));
            if($num_1 !== FALSE)
            {
                $return['id_message'] = substr($message, 0, $num_1);
                $return['submit_date'] = get_time_from_state(substr($message, $num_1+9, 10));
                $return['done_date'] = get_time_from_state(substr($message, $num_1+19, 10));
                $return['stat'] = substr($message, $num_1+29, 7);
                $return['err'] = substr($message, $num_1+36, 3);
            }
        }
        return $return;
    }

    function get_time_from_state($time)
    {
        if(strlen($time) == 10)
        {
            $time = mktime(substr($time, 6, 2), substr($time, 8, 2), 0, substr($time, 2, 2), substr($time, 4, 2), '20'.substr($time, 0, 2));
            $time = date('Y-m-d H:i:s', $time);
            return $time;
        }
        else
            return FALSE;
    }

    function get_state_from_short_name($state)
    {
        switch($state)
        {
            case 'ENROUTE':
                $state_text = 0x1;
                break;
            case 'DELIVRD':
                $state_text = 0x2;
                break;
            case 'DELIVER':
                $state_text = 0x2;
                break;
            case 'EXPIRED':
                $state_text = 0x3;
                break;
            case 'DELETED':
                $state_text = 0x4;
                break;
            case 'UNDELIV':
                $state_text = 0x5;
                break;
            case 'ACCEPTD':
                $state_text = 0x6;
                break;
            case 'UNKNOWN':
                $state_text = 0x7;
                break;
            case 'REJECTD':
                $state_text = 0x8;
                break;
            default:
                $state_text = FALSE;
        }
        return $state_text;
    }

    function get_cookie($name, $type = 'get')
    {
        global $_GET, $_POST;
        if($type == 'get')
            $internal = $_GET;
        else
            $internal = $_POST;
        if(isset($internal[$name]) || isset($internal['array_' . $name]))
        {
            if(is_array($internal[$name]))
            {
                foreach($internal[$name] as $val)
                {
                    if($val !== '')
                    {
                        $set .= $val . ';';
                        $return[] = $val;
                    }
                }
                $name = 'array_' . $name;
                setcookie($name, $set, time() + 30758400, '/');
            }
            elseif(isset($internal[$name]))
            {
                $return = $internal[$name];
                setcookie($name, $return, time() + 30758400, '/');
            }
            else
            {
                $return = '';
                $name = 'array_' . $name;
                setcookie($name, '', time() - 30758400, '/');
            }
        }
        else
        {
            global $_COOKIE;
            if(isset($_COOKIE['array_' . $name]))
            {
                $arr = explode(';', $_COOKIE['array_' . $name]);
                foreach($arr as $val)
                {
                    if($val !== '')
                        $return[] = $val;
                }
            }
            else
                $return = $_COOKIE[$name];
        }
        return $return;
    }

    function num_sms_all()
    {
        global $num_sms_control, $num_turn_time, $num_sms_send,$num_sms_wait,$num_sms_stop,$num_error_sms;
        return $num_sms_send + $num_turn_time + $num_sms_control+$num_sms_stop+$num_sms_wait+$num_error_sms;
    }

     function num_sms_eche_all($send_sms)
     {
         global $error_page,$no_sms,$num_sms_control, $num_turn_time, $num_sms_send,$num_sms_wait,$num_sms_stop,$num_error_sms;

        for($i=0; $i<3; $i++)
        {
            if(isset($send_sms[$i]))
            {
                if(is_array($error = get_error($send_sms[$i])))
                {
                    if(isset($error['fatal_error']))
                    {
                        $error_page[] = array('ERROR', $error['fatal_error']);
                        $no_sms = 1;
                        if($send_sms[$i]['count_sms']>1)
                            $num_error_sms+=$send_sms[$i]['count_sms'];
                        else
                            $num_error_sms++;
                        return FALSE;
                    }
                    else
                        set_error(array('MESSAGE', '+' . $phone_ . ' ' . $error['error']));
                }
                elseif($send_sms[$i]['id_control_text'])
                    $num_sms_control += $send_sms[$i]['count_sms'];
                elseif($send_sms[$i]['turn_time'])
                    $num_turn_time += $send_sms[$i]['count_sms'];
                elseif($send_sms['error'] == 'stop_phone')
                    $num_sms_stop+=$send_sms[$i]['count_sms'];
                elseif($send_sms['error'] == 'sms_wait_payment')
                    $num_sms_wait+=$send_sms[$i]['count_sms'];
                elseif($send_sms['error'] == 'control_text')
                    $num_sms_control+=$send_sms[$i]['count_sms'];
                else
                    $num_sms_send += $send_sms[$i]['count_sms'];
            }
        }
        return TRUE;
     }
    function send_sms_phone($phone, $local, $gradual, $local_time, $id_user, $all_originator, $control_text, $id_partner, $originator, $flash_sms, $text_sms, $text_wap, $link_wap, $name_vcard, $phone_cell_vcard, $phone_work_vcard, $phone_fax_vcard, $e_mail_vcard, $url_vcard, $position_vcard, $organization_vcard, $address_post_office_box_vcard, $address_street_vcard, $address_city_vcard, $address_region_vcard, $address_postal_code_vcard, $address_country_vcard, $additional_vcard, $name_delivery, $vp_hour, $vp_minute, $priority)
    {
        global $phones_send, $error_page, $no_sms, $num_sms_control, $num_turn_time, $num_sms_send,$global_languages,$num_sms_wait,$num_sms_stop,$num_control_text;
        $phone_array = get_array_phone($phone);
        if(is_array($phone_array))
        {
            foreach($phone_array as $phone_)
            {
                $phone_ = get_phone($phone_);
                if($phone_)
                {
                    if(isset($phones_send[$phone_]))
                        set_error(array('MESSAGE', '+' . $phone_ . ' '.$global_languages['NUMBER_OCCURS_TWICE']));
                    elseif(isset($phones_stop[$phone_]))
                        set_error(array('MESSAGE', '+' . $phone_ . ' '.$global_languages['NUMBER_EXLUDED_FROM_SEND_BECAUSE_STOP_LIST']));
                    else
                    {
                        $phones_send[$phone_] = 1;
                        $date_send = get_gradual($phone_, $local, $gradual, $local_time);
                        $send_sms = send_three_sms($id_user, $all_originator, $control_text, $id_partner, $phone_, $originator, $flash_sms, $text_sms, $text_wap, $link_wap, $name_vcard, $phone_cell_vcard, $phone_work_vcard, $phone_fax_vcard, $e_mail_vcard, $url_vcard, $position_vcard, $organization_vcard, $address_post_office_box_vcard, $address_street_vcard, $address_city_vcard, $address_region_vcard, $address_postal_code_vcard, $address_country_vcard, $additional_vcard, $date_send[0], $date_send[1], $name_delivery, $vp_hour, $vp_minute, $priority);
                        if(num_sms_eche_all($send_sms)===FALSE)
                            return FALSE;
                    }
                }
            }
        }
        return TRUE;
    }

    function send_get($url)
    {
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_CRLF, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);   
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, false);   
        curl_setopt($ch, CURLOPT_URL, $url);
        $result = curl_exec($ch);
        $res = $result;
        curl_close($ch);
        return $res;
    }

    function set_works_smpp($id_aggregating)
    {
        global $SQL;
        $query = 'update sms_aggregating set works="1" where id_aggregating="' . $id_aggregating . '"';
        $SQL->query($query);
        return TRUE;
    }

    function set_hidden($name, $value)
    {
        global $tpl_module;
        if(is_array($value))
        {
            foreach($value as $k => $v)
            {
                $tpl_module->get('HIDDEN', array('NAME', 'VALUE'), array($name.'['.$k.']', $v), 1);
            }
        }
        else
            $tpl_module->get('HIDDEN', array('NAME', 'VALUE'), array($name, $value), 1);
    }

    function set_background($name, $value, $action = '')
    {
        global $SQL, $global_id_background;
        if($action)
        {
            $query = 'insert into sms_background set type="' . $action . '"';
            $SQL->query($query);
            $global_id_background = $SQL->insert_id();
        }
        if(is_array($value))
        {
            foreach($value as $k1 => $v1)
            {
                if(is_array($v1))
                {
                    foreach($v1 as $k2 => $v2)
                    {
                        insert_background($name, $v2, $k1, $k2);
                    }
                }
                else
                    insert_background($name, $v1, $k1);
            }
        }
        else
            insert_background($name, $value);
    }

    function insert_background($name, $value, $array_1 = '', $array_2 = '')
    {
        global $SQL, $global_id_background;
        $query = 'insert into sms_background_value set name="' . $name . '", value="' . $value . '", array_1="' . $array_1 . '", array_2="' . $array_2 . '", id_background="' . $global_id_background . '"';
        $SQL->query($query);
    }

    function get_background($id_background)
    {
        global $SQL;
        $query = 'select name, value, array_1, array_2 from sms_background_value where id_background="' . $id_background . '" order by name';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++)
        {
            $name = $SQL->result($result, $i, 'name');
            if(!isset(${$name}))
                global ${$name};
            $value = $SQL->result($result, $i, 'value');
            $array_1 = $SQL->result($result, $i, 'array_1');
            $array_2 = $SQL->result($result, $i, 'array_2');
            if($array_2 !== '' && $array_1 !== '')
                ${$name}[$array_1][$array_2] = $value;
            elseif($array_1 !== '')
                ${$name}[$array_1] = $value;
            else
                ${$name} = $value;
        }
    }

    function set_message($id_user, $message, $type = 'text', $href='')
    {
        global $SQL, $global_id_background;
        if(is_array($message))
        {
            $run = $message['run'];
            $message = $message['message'];
        }else
            $run = '';
        $query = 'insert into messages set message="' . addslashes($message) . '", run="' . addslashes($run) . '", id_user="' . $id_user . '"';
        $SQL->query($query);
        $id_message = $SQL->insert_id();
        if($type == 'progress')
        {
            $error_message = get_progress_message($id_message, $global_id_background, $message,$href);
            $query = 'update messages set run="' . addslashes($error_message['run']) . '", message="' . addslashes($error_message['message']) . '" where id_message="' . $id_message . '"';
            $SQL->query($query);
        }
    }

    function get_name_file($dir, $ext)
    {
        for($i=0; $i<100; $i++)
        {
            $rnd = rand(1, 1000000000);
            $file = $dir . $rnd . '.' . $ext;
            if(!file_exists($file))
                return $rnd;
        }
        return FALSE;
    }

    function check_answer($id_phone, $id_answer)
    {
        global $SQL;
        $query = 'select * from sms_receive_sms_answer where id_answer="' . $id_answer . '" and id_phone="' . $id_phone . '"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
            return TRUE;
        else
            return FALSE;
    }

    function check_reveive_phone($id_user, $id_phone)
    {
        global $SQL;
        $query = 'select * from sms_receive_sms where id_phone="' . $id_phone . '" and id_user="' . $id_user . '"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
            return TRUE;
        else
            return FALSE;
    }

    function set_num_sms($id_background, $num_type_sms, $rest_array, $ready)
    {
        $rest = 0;
        if(is_array($rest_array))
        {
            foreach($rest_array as $num)
            {
                $rest += $num;
            }
        }
        $rest *= $num_type_sms;
        set_progress_background($id_background, $rest, $ready);
    }

    function set_progress_background($id_background, $rest, $ready)
    {
        global $SQL;
        $query = 'update sms_background set ready="' . $ready . '", rest="' . $rest . '", state="process" where id_background="' . $id_background . '"';
        $SQL->query($query);
    }

    function get_progress_message($id_message, $id_background, $message, $href='')
    {
        global $version_tpl;
        $tpl_progress = new tpl('module/message_progress_' . $version_tpl);
        $tpl_progress->get('PROGRESS', array('ID_BACKGROUND', 'MESSAGE', 'ID_MESSAGE'), array($id_background, $message, $id_message));
        $return['message'] = $tpl_progress->content;
        $return['run'] = 'set_progress(' . $id_background . ',"'.$href.'");';
        return $return;
    }

    function get_js_value($message)
    {
        $message = str_replace('\\', '\\\\', $message);
        $message = str_replace('\'', '\\\'', $message);
        $message = str_replace("\n\r", '', $message);
        $message = str_replace("\r\n", '', $message);
        $message = str_replace("\n", '', $message);
        return $message;
    }

    function insert_update($type, $id)
    {
        global $SQL;
        $query = 'insert into sms_update set type="' . $type . '", id="' . $id . '"';
        $SQL->query($query);
    }

    function get_param_domain_partner($domain_partner)
    {
        global $SQL;
        if($domain_partner == HTTP_SERVER)
            $id_partner = 0;
        else
        {
            $query = 'select id_partner from sms_partners where domain="' . addslashes($domain_partner) . '"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result))
                $id_partner = $SQL->result($result, 0, 'id_partner');
            else
                $id_partner = 0;
        }
        return array('id_partner' => $id_partner);
    }

    function rnd_string($length)
    {
        $string = array('Q', 'W', 'R', 'Y', 'U', 'S', 'D', 'F', 'G', 'J', 'L', 'Z', 'V', 'N', '2', '4', '5', '7', '8', '9');
        $len = count($string) - 1;
        $return = '';
        for($i = 0; $i < $length; $i++)
        {
            $return .= $string[rand(0, $len)];
        }
        return $return;
    }

    function read_file_in_array($tmp_name, $type)
    {
        global $global_languages;
        if($type == 'xls')
        {
            $data = new Spreadsheet_Excel_Reader();
            $data->setOutputEncoding('UTF-8');
            $data->read($tmp_name);
            if($data->_ole->error == 1)
                $return = $global_languages['WRONG_XLS_FILE'];
            else
            {
                $return = array();
                $count = count($data->sheets);
                for($sheet=0;$sheet<$count;$sheet++)
                {
                    for($row=1;$row<=$data->sheets[$sheet]["numRows"];$row++)
                    {
                        $array = array();
                        for($col=0; $col<$data->sheets[$sheet]["numCols"]; $col++)
                        {
                            $array[$col] = $data->sheets[$sheet]["cells"][$row][$col+1];
                        }
                        $return[] = $array;
                    }
                }
            }
        }
        elseif($type == 'xlsx')
        {
            $xlsx = new SimpleXLSX($tmp_name);
            $num_sheets = $xlsx->sheetsCount();
            for($sheet=1;$sheet<=$num_sheets;$sheet++)
            {
                list($num_cols, $num_rows) = $xlsx->dimension($sheet);
                $rows = $xlsx->rows($sheet);
                for($j=0; $j<$num_rows; $j++)
                {
                    $return[] = $rows[$j];
                }
            }
        }
        elseif($type == 'csv')
        {
            $fp = fopen($tmp_name, 'r');
            while(($data = fgetcsv($fp, 3072000, ';', '"', '"')) !== FALSE)
            {
                $num = count($data);
                for($c=0; $c < $num; $c++)
                {
                    $data[$c] = iconv('Windows-1251', 'UTF-8', $data[$c]);
                }
                $return[] = $data;
            }
            fclose ($fp);
        }
        return $return;
    }

    function admin_global_languages()
    {
        global $current_language;
        current_language();
        return get_global_languages($current_language);
    }

    function get_global_languages($language)
    {
        global $SQL;
        $return = array();
        $query = 'select name, value from global_languages where language="' . $language . '"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++)
        {
            $name = $SQL->result($result, $i, 'name');
            $value = $SQL->result($result, $i, 'value');
            $return[$name] = $value;
        }
        return $return;
    }

    function get_all_languages()
    {
        global $SQL;
        $return = array();
        $query = 'select name, value, language from global_languages order by language';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++)
        {
            $language=$SQL->result($result, $i, 'language');
            $name = $SQL->result($result, $i, 'name');
            $value = $SQL->result($result, $i, 'value');
            $return[$language][$name] = $value;
        }
        return $return;
    }

    function echo_languages($tpl, $href = '')
    {
        global $SQL;
        $query = 'select  name, type from languages';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++)
        {
            $language = $SQL->result($result, $i, 'name');
            $type = $SQL->result($result, $i, 'type');
            $tpl->get('LANGUAGE', array('LANGUAGE', 'HREF','TYPE'), array($language, $href,$type), 1);
        }
    }

    function get_array_languages()
    {
        global $SQL, $global_languages;
        $query = 'select name from languages';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        $return=array();
        for($i=0; $i<$num; $i++)
        {
            $language = $SQL->result($result, $i, 'name');
            $return[]=array('id'=>$language, 'name'=>$global_languages[strtoupper($language)]);
        }
        return $return;
    }

    function get_href($var, $num)
    {
        $href = '';
        $count = count($href);
        for($i=$num; $i<$count; $i++)
        {
            $href . '/' . $var[$i];
        }
        if($href)
            $href .= '.html';
        return $href;
    }

    function get_increment($name)
    {
        global $SQL;
        $query = 'lock tables sms_increment write';
        $SQL->query($query);
        $query = 'select id from sms_increment where name="' . $name . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id = $SQL->result($result, 0, 'id') + 1;
            $query = 'update sms_increment set id=' . $id . ' where name="' . $name . '"';
            $SQL->query($query);
        }
        $query = 'unlock tables';
        $SQL->query($query);
        return $id;
    }

    function get_increment_account($name,$id_auth_partner,$id_company=FALSE)
    {
        global $SQL;
        if($id_company)
        {
            $query = 'select current_account from sms_companys where id_company="'.$id_company.'"';
            $result_value = $SQL->query($query);
            if($res = $SQL->fetch_assoc($result_value))
            {
                $query = 'UPDATE sms_companys SET current_account="'.($res['current_account']+1).'"  where id_company="'.$id_company.'"';
                $result = $SQL->query($query);
                return $res['current_account'];
            }
        }
        $query = 'select sav.id_adjustment,sav.value from sms_adjustment_value as sav INNER JOIN (SELECT id_adjustment, adjustment  FROM sms_adjustment where adjustment="'.$name.'")as sa ON (sa.id_adjustment=sav.id_adjustment) where sav.id_partner="' . $id_auth_partner . '"';
        $result_value = $SQL->query($query);
        if($num_rows_value=$SQL->num_rows($result_value))
        {
            $id_adjustment=$SQL->result($result_value, 0, 'sav.id_adjustment');
            $value = $SQL->result($result_value, 0, 'sav.value');
            $new_value = $value + 1;
            $query = 'update sms_adjustment_value set value="' . $new_value . '" where id_adjustment="' . $id_adjustment . '" and id_partner="' . $id_auth_partner . '"';
        }
        else
        {
            $value=1;
            $new_value=2;
            $query = 'insert into sms_adjustment_value set value="' . $new_value . '", id_adjustment="' . $id_adjustment . '", id_partner="' . $id_auth_partner . '"';
        }
        $SQL->query($query);
        return $value;
    }

    function get_new_account($id_user,$id_partner,$sum)
    {
        global $SQL;
        $query = 'SELECT id_company, id_manager, login FROM sms_send_users where id_user="'.$id_user.'"';
        $result = $SQL->query($query);
        if(!$result)exit();
        $res = $SQL->fetch_assoc($result);
        $account = get_increment_account('CURRENT_ACCOUNT',$id_partner,$res['id_company']);
        $query = 'INSERT INTO sms_pay_account SET account="'.$account.'", id_user="'.$id_user.'", id_partner="'.$id_partner.'", id_company="'.$res['id_company'].'", sum="'.$sum.'", date="' . date('Y-m-d') . '"';
        $SQL->query($query);
        $insert_id = $SQL->insert_id();
        if($id_partner)
        {
            $choice = 'select domain from sms_partners where id_partner="' . $id_partner . '"';
            $result = $SQL->query($choice);
            if($SQL->num_rows($result))
                $domain = $SQL->result($result, 0, 'domain');
            else
                $domain = HTTP_SERVER;
        }
        else
            $domain = HTTP_SERVER;
        $mail_message = 'Клиент '.$res["login"].', выписал счет на сумму '.$sum.' руб. Ссылка на счет: https://'.$domain.'/lkadm/section.php?id_page=72&action=view&id_account='.$insert_id;
        if($res["id_manager"]){
            send_mail_manager($res["id_manager"], 'Выписан счет ' . $res["login"], $mail_message);
        } else {
            send_mail_partner($id_partner, 'Выписан счет ' . $res["login"], $mail_message);
        }
        return $insert_id;
    }

    function get_account($id_account)
    {
        global $SQL;
        $query = 'SELECT * FROM sms_pay_account WHERE id_account="'.$id_account.'"';
        $result = $SQL->query($query);
        return $SQL->fetch_assoc($result);
    }

    function exist_account($id_user)
    {
        global $SQL;
        $query = 'select count(*) as count from sms_pay_account where id_user="' . $id_user . '" order by date desc';
        $result = $SQL->query($query);
        if($res = $SQL->fetch_assoc($result))
            return $res['count'];
        return 0;
    }

    function add_hexdec($num)
    {
        if(substr($num, 0, 2) == '0x')
            $num = hexdec($num);
        return $num;
    }
    function change_User_Partnenr_From($ID_User,$ID_Part,$From)
    {
        global $SQL;
        $query = 'update '.$From.' set id_partner=' . $ID_Part . ' where id_user=' . $ID_User ;
        $result=$SQL->query($query);
    }
    function change_User_Partnenr($User,$Part)
    {
        change_User_Partnenr_From($User,$Part,'sms_send_users');
        change_User_Partnenr_From($User,$Part,'sms_incoming');
        change_User_Partnenr_From($User,$Part,'sms_send_trafic');
        change_User_Partnenr_From($User,$Part,'sms_send_trafic_archive');
        change_User_Partnenr_From($User,$Part,'sms_send_trafic_report');
        change_User_Partnenr_From($User,$Part,'sms_stat');
        change_User_Partnenr_From($User,$Part,'sms_turn');
        change_User_Partnenr_From($User,$Part,'sms_turn_time');
    }

   function add_reserve_aggregating($id_aggregating)
   {
        global $SQL;
        $query = 'UPDATE sms_aggregating set cope="0" where id_aggregating="'.$id_aggregating.'" and type="SMPP"' ;
        $result =$SQL->query($query);
   }

   function delete_reserve_aggregating($id_aggregating)
   {
        global $SQL;
        $query = 'UPDATE sms_aggregating set cope="1" where id_aggregating="'.$id_aggregating.'"' ;
        $result =$SQL->query($query);
   }

   function set_originator_in_array()
   {
        global $SQL, $originator_array;
        $originator_array = array();
        $query = 'select id_client, MCC, MNC, originator from sms_phone_code_originator';
        $result = $SQL->unbuffered_query($query);
        while($res = $SQL->fetch_assoc($result))
        {
            $originator_array[$res['id_client']][$res['MCC']][$res['MNC']]=$res['originator'];
        }
   }

   function get_originator_from_array($user,$MCC,$MNC)
   {
         global $originator_array;
         if(isset($originator_array[$user][$MCC][$MNC]))
            return $originator_array[$user][$MCC][$MNC];
         elseif(isset($originator_array[$user][$MCC][0]))
            return $originator_array[$user][$MCC][0];
         elseif(isset($originator_array[0][$MCC][$MNC]))
            return $originator_array[0][$MCC][$MNC];
         elseif(isset($originator_array[0][$MCC][0]))
            return $originator_array[0][$MCC][0];
         else
            return FALSE;
   }

   function get_originator_from_array_replace($id_user,$originator,$MCC,$MNC)
   {
        global $SQL;
        $date = date('Y-m-d');
        $query = 'select replacement_originator from sms_replacement_originator where id_user="'.$id_user.'" and originator="'.$originator.'" and MCC="'.$MCC.'" and (MNC="'.$MNC.'" OR MNC=0) and date_start<="'.$date.'" and date_end>="'.$date.'"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
            return $SQL->result($result, 0, 'replacement_originator');
        return FALSE;
   }
   
   function get_parameters($id, $type)
   {
        global $SQL;
            $query = 'select value from parameters where id="' . $id . '" and type="' .$type.'"';
            $result = $SQL->query($query);
            $num = $SQL->num_rows($result);
            if($num)
             return $SQL->result($result, $i, 'value');
           return FALSE;
   }


    function get_array_time_zone($time_zone_all)
   {
       global $global_languages;
       $languages_local_time=array();
      if($time_zone_all)
      {
         $tmp= explode(';',$time_zone_all);
         foreach($tmp as $id )
         {
            if(is_numeric($id))
            {
               $l=(string)$id;
               if($id<0)
                  $l[0]='M';
               else
                  $l='P'.$id;
               $l='UTC_'.$l;
               if($global_languages[$l]) // Проверяем наличия названия часового пояса
                  $languages_local_time[]=array('id' => $id,'name'=>$global_languages[$l]);
            }
         }
      }
      return $languages_local_time;
   }

  function get_translate_phone_code_title($title)
  {
     global $global_languages;
     if($global_languages[$title])
        return $global_languages[$title];
     else
        return $title;
  }
  function get_multi_select_MCC_MNC($id_select= FALSE,$only=FALSE)
  {
      global $SQL, $select_MCC_MNC;
      if(!isset($select_MCC_MNC))
      {
      $query='select if(MNC=0,-1000,sp.priority)as priority,if(priority_ is NULL,sp.priority,priority_) as priority_, CONCAT(MCC,":",MNC)  as id, title, CONCAT(" MCC=",MCC," MNC=",MNC) as name, MNC from sms_phone_code_title as sp LEFT JOIN (Select MCC as MCC_,priority as priority_ FROM sms_phone_code_title where MNC=0) as spct ON(spct.MCC_=sp.MCC) order by priority_ asc,sp.MCC,priority asc, title asc';
      $result = $SQL->query($query);
          $select_MCC_MNC= array();
        while($res=$SQL->fetch_assoc($result))
        {
            if($res['MNC']!=0)
                    $res['name'] = '&nbsp;&nbsp;&nbsp;'.get_translate_phone_code_title($res['title']).$res['name'];
                else
                    $res['name'] = get_translate_phone_code_title($res['title']).$res['name'];
                $select_MCC_MNC[$res['id']]=$res;
            }
      }
if(!isset($select_MCC_MNC)) return '<option value="' . $id_select . '">' . $id_select . '</option>';
if($only)
{
    $option='';
    foreach($id_select as  $key=>$v)
        $option .='<option value="' . $key . '">' . $select_MCC_MNC[$key]['name'] . '</option>';
    return $option;
}
        foreach($select_MCC_MNC as $res)
        {
            if($res['MNC']!=0)
                $option[]=array('id'=>$res['id'], 'name'=>('&nbsp;&nbsp;&nbsp;'.$res['name']));
            else
                $option[]=array('id'=>$res['id'], 'name'=>($res['name']));
        }
     return get_js_value(get_multi_select_from_array($option,$id_select));
  }

  function get_translate_phone_code($title_and_code)
  {
        if(!($l=strpos($title_and_code, ' MCC', $l)))
           if(!($l=strpos($title_and_code, ' MNC', $l)))
               return $title_and_code.$only_code;
        $only_title=substr($title_and_code,0,$l);
        $only_code=substr($title_and_code,$l,strlen($title_and_code));
        $only_title=get_translate_phone_code_title($only_title);
        return $only_title.$only_code;
  }

  function get_comparison($comparison)
  {
     if($comparison=='>=' || $comparison=='<')
        return $comparison;
     return '>=';
  }

    function get_set_from_query($query)
    {
        global $SQL;
       $option = '';
       $result = $SQL->query($query);
        while($res=$SQL->fetch_assoc($result))
        {
            $option .= '"' . $res['id'] . '", ';
         }
        $option = substr($option, 0, -2);
        return $option;
    }

    function get_cache_in_user_bases($id_user)
    {
       global $SQL, $global_in_user_bases;
       if(isset($global_in_user_bases[$id_user]))
          return $global_in_user_bases[$id_user];
       else
       {
          $query_base='SELECT id_base as id FROM sms_base WHERE id_user="'.$id_user.'" ';
          $global_in_user_bases[$id_user]=get_set_from_query($query_base);
          return $global_in_user_bases[$id_user];
       }
    }


    function obj_in_array($xml)
    {
        $array=array();
        $array_obj = (array)$xml;
        $count_k = array();
        foreach($xml as $k=>$v)
        {
            if(!isset($count_k[$k]))
                $count_k[$k] = 0;
            $tmp=&$array[$k][$count_k[$k]];
            if(is_string($array_obj[$k]))
                $tmp['v']=$array_obj[$k];
            elseif(is_string($array_obj[$k][$count_k[$k]]))
                $tmp['v']=$array_obj[$k][$count_k[$k]];
            else
            {
                $res = obj_in_array($v);
                if(count($res)>0)
                    $tmp['v']= $res;
            }
            $atr = (array)$v;
            if(isset($atr['@attributes']))
                $tmp['a']=$atr['@attributes'];
            $count_k[$k]++;
        }

        return $array;
    }

    function XML_in_array($in)
    {

        $xml = new SimpleXMLElement($in);
        return obj_in_array($xml);
    }

    function copy_tpl($dir_from, $dir_to)
    {
        @mkdir($dir_to, 0777);
        @mkdir($dir_to . 'arrangement', 0777);
        @mkdir($dir_to . 'module', 0777);

        copy($dir_from . 'account.tpl', $dir_to . 'account.tpl');
        copy($dir_from . 'header.tpl', $dir_to . 'header.tpl');
        copy($dir_from . 'body.tpl', $dir_to . 'body.tpl');
        copy($dir_from . 'header_window_body.tpl', $dir_to . 'header_window_body.tpl');
        copy($dir_from . 'settings.js.tpl', $dir_to . 'settings.js.tpl');
        copy($dir_from . 'settings.css.tpl', $dir_to . 'settings.css.tpl');
        copy($dir_from . 'window.tpl', $dir_to . 'window.tpl');
        copy($dir_from . 'receipt.tpl', $dir_to . 'receipt.tpl');
        copy($dir_from . 'arrangement/8.tpl', $dir_to . 'arrangement/8.tpl');
        copy($dir_from . 'module/auth.tpl', $dir_to . 'module/auth.tpl');
        copy($dir_from . 'module/multi_2text_2title_file_1.tpl', $dir_to . 'module/multi_2text_2title_file_1.tpl');
        copy($dir_from . 'module/cabinet_sms_load_buffer_0.tpl', $dir_to . 'module/cabinet_sms_load_buffer_0.tpl');
        copy($dir_from . 'module/cabinet_sms_activation_0.tpl', $dir_to . 'module/cabinet_sms_activation_0.tpl');
        copy($dir_from . 'module/cabinet_sms_all_stats_0.tpl', $dir_to . 'module/cabinet_sms_all_stats_0.tpl');
        copy($dir_from . 'module/cabinet_sms_incoming_0.tpl', $dir_to . 'module/cabinet_sms_incoming_0.tpl');
        copy($dir_from . 'module/cabinet_sms_incoming_add_in_base_0.tpl', $dir_to . 'module/cabinet_sms_incoming_add_in_base_0.tpl');
        copy($dir_from . 'module/cabinet_sms_incoming_list_0.tpl', $dir_to . 'module/cabinet_sms_incoming_list_0.tpl');
        copy($dir_from . 'module/cabinet_sms_load_xls_1_0.tpl', $dir_to . 'module/cabinet_sms_load_xls_1_0.tpl');
        copy($dir_from . 'module/cabinet_sms_load_xls_2_0.tpl', $dir_to . 'module/cabinet_sms_load_xls_2_0.tpl');
        copy($dir_from . 'module/cabinet_sms_originator_0.tpl', $dir_to . 'module/cabinet_sms_originator_0.tpl');
        copy($dir_from . 'module/cabinet_sms_pattern_0.tpl', $dir_to . 'module/cabinet_sms_pattern_0.tpl');
        copy($dir_from . 'module/cabinet_sms_pattern_edit_0.tpl', $dir_to . 'module/cabinet_sms_pattern_edit_0.tpl');
        copy($dir_from . 'module/cabinet_sms_pay_0.tpl', $dir_to . 'module/cabinet_sms_pay_0.tpl');
        copy($dir_from . 'module/cabinet_sms_phone_0.tpl', $dir_to . 'module/cabinet_sms_phone_0.tpl');
        copy($dir_from . 'module/cabinet_sms_phone_base_0.tpl', $dir_to . 'module/cabinet_sms_phone_base_0.tpl');
        copy($dir_from . 'module/cabinet_sms_phone_base_add_0.tpl', $dir_to . 'module/cabinet_sms_phone_base_add_0.tpl');
        copy($dir_from . 'module/cabinet_sms_phone_base_add_phones_0.tpl', $dir_to . 'module/cabinet_sms_phone_base_add_phones_0.tpl');
        copy($dir_from . 'module/cabinet_sms_phone_base_edit_0.tpl', $dir_to . 'module/cabinet_sms_phone_base_edit_0.tpl');
        copy($dir_from . 'module/cabinet_sms_phone_edit_0.tpl', $dir_to . 'module/cabinet_sms_phone_edit_0.tpl');
        copy($dir_from . 'module/cabinet_sms_profile_0.tpl', $dir_to . 'module/cabinet_sms_profile_0.tpl');
        copy($dir_from . 'module/cabinet_sms_report_0.tpl', $dir_to . 'module/cabinet_sms_report_0.tpl');
        copy($dir_from . 'module/cabinet_sms_rest_0.tpl', $dir_to . 'module/cabinet_sms_rest_0.tpl');
        copy($dir_from . 'module/cabinet_sms_send_sms_0.tpl', $dir_to . 'module/cabinet_sms_send_sms_0.tpl');
        copy($dir_from . 'module/cabinet_sms_set_phone_0.tpl', $dir_to . 'module/cabinet_sms_set_phone_0.tpl');
        copy($dir_from . 'module/cabinet_sms_stats_0.tpl', $dir_to . 'module/cabinet_sms_stats_0.tpl');
        copy($dir_from . 'module/cabinet_sms_stop_0.tpl', $dir_to . 'module/cabinet_sms_stop_0.tpl');
        copy($dir_from . 'module/cabinet_sms_stop_edit_0.tpl', $dir_to . 'module/cabinet_sms_stop_edit_0.tpl');
        copy($dir_from . 'module/cabinet_sms_turn_time_0.tpl', $dir_to . 'module/cabinet_sms_turn_time_0.tpl');
        copy($dir_from . 'module/cabinet_sms_user_report_0.tpl', $dir_to . 'module/cabinet_sms_user_report_0.tpl');
        copy($dir_from . 'module/cabinet_sms_cleaning_phone_0.tpl', $dir_to . 'module/cabinet_sms_cleaning_phone_0.tpl');
        copy($dir_from . 'module/menu_sms_0.tpl', $dir_to . 'module/menu_sms_0.tpl');
        copy($dir_from . 'module/offer_0.tpl', $dir_to . 'module/offer_0.tpl');
        copy($dir_from . 'module/registration_0.tpl', $dir_to . 'module/registration_0.tpl');
        copy($dir_from . 'module/registration_restoration_0.tpl', $dir_to . 'module/registration_restoration_0.tpl');
        copy($dir_from . 'module/load_parent_0.tpl', $dir_to . 'module/load_parent_0.tpl');
        copy($dir_from . 'module/text_0.tpl', $dir_to . 'module/text_0.tpl');
        copy($dir_from . 'module/help_0.tpl', $dir_to . 'module/help_0.tpl');
        copy($dir_from . 'module/help_list_0.tpl', $dir_to . 'module/help_list_0.tpl');
        copy($dir_from . 'module/text_title_0.tpl', $dir_to . 'module/text_title_0.tpl');
        copy($dir_from . 'module/cabinet_sms_check_send_sms_0.tpl', $dir_to . 'module/cabinet_sms_check_send_sms_0.tpl');
        copy($dir_from . 'module/sms_title_text_partner_0.tpl', $dir_to . 'module/sms_title_text_partner_0.tpl');
        copy($dir_from . 'module/sms_title_text_partner_1.tpl', $dir_to . 'module/sms_title_text_partner_1.tpl');
        copy($dir_from . 'module/sms_title_text_partner_2.tpl', $dir_to . 'module/sms_title_text_partner_2.tpl');
        copy($dir_from . 'module/cabinet_sms_price_0.tpl', $dir_to . 'module/cabinet_sms_price_0.tpl');
        copy($dir_from . 'module/cabinet_sms_data_0.tpl', $dir_to . 'module/cabinet_sms_data_0.tpl');
        copy($dir_from . 'module/message_progress_0.tpl', $dir_to . 'module/message_progress_0.tpl');
        copy($dir_from . 'module/cabinet_sms_wait_0.tpl', $dir_to . 'module/cabinet_sms_wait_0.tpl');
        copy($dir_from . 'module/cabinet_sms_account_0.tpl', $dir_to . 'module/cabinet_sms_account_0.tpl');
        copy($dir_from . 'module/cabinet_sms_all_stat_0.tpl', $dir_to . 'module/cabinet_sms_all_stat_0.tpl');
        copy($dir_from . 'module/news_0.tpl', $dir_to . 'module/news_0.tpl');
        copy($dir_from . 'module/news_more_0.tpl', $dir_to . 'module/news_more_0.tpl');
        copy($dir_from . 'module/cabinet_sms_find_phone_0.tpl', $dir_to . 'module/cabinet_sms_find_phone_0.tpl');
        copy($dir_from . 'module/error_0', $dir_to . 'module/error_0');
        return TRUE;
    }


    function pay_file_add($name,$id=FALSE)
    {
        global $SQL, $_FILES;
        if(!isset($_FILES[$name]))
            return '';
        $file = &$_FILES[$name];
        $tmp_file = &$file['tmp_name'];
        if(file_exists($tmp_file))
        {
            $type = get_type_file($file['name']);
            if(!$id)
            {
                $query = 'INSERT into sms_file Set name="'.$name.'", type="'.$type.'"';
                $SQL->query($query);
                $id = $SQL->insert_id();
                $return = $name.'="'.$id.'",';
            } else
            {
                $return ='';
                $query = 'UPDATE sms_file Set name="'.$name.'", type="'.$type.'" WHERE id_file="'.$id.'"';
                $SQL->query($query);
            }
            $file_name = '../pay/' . $id . '.'.$type;
            if(file_exists($file_name))
                unlink($file_name);
            copy($tmp_file, $file_name);
            return $return;
        }
        return '';
    }
    // Возвращает ссылку файла
    function pay_file_get($id)
    {
        global $SQL;
        $query = 'SELECT * FROM sms_file where id_file="'.$id.'"';
        if(!$result = $SQL->query($query)) return FALSE;
        if(!$res = $SQL->fetch_assoc($result)) return FALSE;
        return $id.'.'.$res['type'];
    }

    function send_information($id_user,$limit_price)
    {
        if(!$limit_price || !$id_user) return FALSE;
        set_param_send_sms();
        $user = get_param_id_user($id_user);
        $param_site = get_param_site($user['id_partner']);
        $text = $param_site['INFORM_PAY_SMS'];
        if($text)
        {
            $text = str_replace('{SUM}',$limit_price,$text);
            if($user['name'])
                $text = str_replace('{NAME}',$user['name'],$text);
            else
                $text = str_replace('{NAME}',$user['login'],$text);
        }
        else
            $text = 'Вам на счёт зачислена сумма: '.$limit_price;
        return send_sms(ID_SMS_SITE, 1, 0, 0, $user['phone'], $param_site['SENDER_SERVICE_SMS'], 'sms', $text);

    }
    function get_company_by_id($id_company)
    {
        global $SQL, $GLOBAL_CACHE_DATA;
        if(isset($GLOBAL_CACHE_DATA['id_company']))
        {
            if($GLOBAL_CACHE_DATA['id_company'][$id_company])
                return $GLOBAL_CACHE_DATA['id_company'][$id_company];
            else
                return '';
        }else
        {
            $query = 'select id_company as id, name from sms_companys where id_partner="' . $id_partner_manager . '" order by name';
            if(!$result = $SQL->query($query)) return '';
            $GLOBAL_CACHE_DATA['id_company'] = array();
            while($res = $SQL->fetch_assoc($result))
            {
                $GLOBAL_CACHE_DATA['id_company'][$res['id']] = $res['name'];
            }
            if($GLOBAL_CACHE_DATA['id_company'][$id_company])
                return $GLOBAL_CACHE_DATA['id_company'][$id_company];
            else
                return '';
        }
    }

    function query_dump($query)
    {
        echo '<!--';
        var_dump($query);
        echo '-->';
    }
    function get_value_from_id($key,$v,$table)
    {
        global $SQL, $GLOBAL_CACHE_DATA;
        if(isset($GLOBAL_CACHE_DATA[$table]))
        {
            return $GLOBAL_CACHE_DATA[$table][$v];
        }else
        {
            $query = 'select '.$key.' as id, name from '.$table;
            if(!$result = $SQL->query($query)) return '';
            while($res = $SQL->fetch_assoc($result))
            {
                $GLOBAL_CACHE_DATA[$table][$res['id']] = $res['name'];
            }
            if($GLOBAL_CACHE_DATA[$table][$v])
                return $GLOBAL_CACHE_DATA[$table][$v];
            else
                return '';
        }
    }
    function change_aggregation($in_id_aggregating, $out_id_aggregating = 0)
    {
        global $SQL;
        do{
            $SQL->connect();
            if($in_id_aggregating)
                $query = 'select st.id_turn, st.id_user, st.phone, st.count_sms, st.id_partner,st.MCC,st.MNC, st.type_sms, st.originator from sms_turn as st where st.id_aggregating="'.$in_id_aggregating.'" limit 5000';
            else
                $query = 'select st.id_turn, st.id_user, st.phone, st.count_sms, st.id_partner,st.MCC,st.MNC, st.type_sms, st.originator from sms_turn as st where (st.id_aggregating="0" or st.id_aggregating in (SELECT id_aggregating FROM sms_aggregating where works="0" or switch="off")) limit 5000';
            //$query = 'select id_turn, id_user, phone, count_sms, id_partner, MCC, MNC, type_sms, originator from sms_turn where st.id_aggregating="0" and st.id_user=ssu.id_user';
            $result = $SQL->query($query);
            $num = $SQL->num_rows($result);
            $change = 0;
            for($i=0; $i<$num; $i++)
            {
                $id_turn = $SQL->result($result, $i, 'st.id_turn');
                $id_user = $SQL->result($result, $i, 'st.id_user');
                $count_sms = $SQL->result($result, $i, 'st.count_sms');
                $id_partner = $SQL->result($result, $i, 'st.id_partner');
                $MCC = $SQL->result($result, $i, 'st.MCC');
                $MNC = $SQL->result($result, $i, 'st.MNC');
                $type_sms = $SQL->result($result, $i, 'st.type_sms');
                $originator = $SQL->result($result, $i, 'st.originator');
                if($out_id_aggregating)
                    $new_id_aggregating = $out_id_aggregating;
                else
                    $new_id_aggregating = get_id_aggregating($id_user, $id_partner,  $MCC,$MNC,$type_sms, $originator, $count_sms);
                if(($new_id_aggregating || $in_id_aggregating)&&($new_id_aggregating!=$in_id_aggregating))
                {
                    $query = 'update sms_turn set id_aggregating="' . $new_id_aggregating . '" where id_turn="' . $id_turn . '"';
                    $SQL->query($query);
                    $change++;
                }
            }
            $SQL->free_result($result);
        }while($num && $change);
    }


    function get_param_aggregating($id_aggregating)
    {
        global $SQL;
        $query = 'select sql_cache *  from sms_aggregating where id_aggregating="' . $id_aggregating . '"';
        $result = $SQL->query($query);
        if($res = $SQL->fetch_assoc($result))
            return $res;
        return FALSE;
    }

    function get_param_id_base($id_base)
    {
        global $SQL;
        $query = 'select sql_cache * from sms_base where id_base="' . $id_base . '"';
        $result = $SQL->query($query);
        if($param_base=$SQL->fetch_assoc($result))
        {
            if($param_base['show_region'])
                $return['region'] = $param_base['name_region'];
            if($param_base['show_operator'])
                $return['operator'] = $param_base['name_operator'];
            if($param_base['show_surname'])
                $return['surname'] = $param_base['name_surname'];
            if($param_base['show_name'])
                $return['name'] = $param_base['name_name'];
            if($param_base['show_patronymic'])
                $return['patronymic'] = $param_base['name_patronymic'];
            if($param_base['show_date_birth'])
                $return['date_birth'] = $param_base['name_date_birth'];
            if($param_base['show_male'])
                $return['male'] = $param_base['name_male'];
            if($param_base['show_addition_1'])
                $return['addition_1'] = $param_base['name_addition_1'];
            if($param_base['show_addition_2'])
                $return['addition_2'] = $param_base['name_addition_2'];
            if($param_base['show_addition_3'])
                $return['addition_3'] = $param_base['name_addition_3'];
            if($param_base['show_addition_4'])
                $return['addition_4'] = $param_base['name_addition_4'];
            if($param_base['show_addition_5'])
                $return['addition_5'] = $param_base['name_addition_5'];
            return $return;
        }
        else
            return FALSE;
    }

    function make_dir($dir)
    {
        $dir = explode('/', $dir);
        if($dir[0] === '')
            $patch = '/';
        foreach($dir as $d)
        {
            if($d)
            {
                $patch .= $d . '/';
                if(!@file_exists($patch))
                    @mkdir($patch, 0777);
            }
        }
    }

    function reload_tpl_partners()
    {
        global $SQL;
        $query = 'select id_partner, name, domain, login, e_mail, block from sms_partners';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++)
        {
            $id_partner = $SQL->result($result, $i, 'id_partner');

            $dir_to = SERVER_ROOT . '/tpl/partners/' . $id_partner . '/';
            $dir_from = SERVER_ROOT . '/tpl/tpl_partners/';
            copy_tpl($dir_from, $dir_to);
        }
    }

    function set_convertion_currency()
    {
        global $SQL, $convertion_currency;
        $query = 'select id_currency from sms_currency where short_title="RUR"';
        $result = $SQL->query($query);
        $res = $SQL->fetch_assoc($result);
        $convertion_currency[$res['id_currency']] = 1;

        $query = 'select id_currency, max(date) as date from sms_currency_conversion group by id_currency';
        $result = $SQL->query($query);
        while($res = $SQL->fetch_assoc($result))
        {
            $query = 'select value from sms_currency_conversion where id_currency="' . $res['id_currency'] . '" and date="' . $res['date'] . '"';
            $result_currency = $SQL->query($query);
            while($res_currency = $SQL->fetch_assoc($result_currency))
            {
                $convertion_currency[$res['id_currency']] = $res_currency['value'];
            }
        }
    }
    function get_convertion_currency($sum, $id_currency_have, $id_currency_need)
    {
        if($sum)
        {
            global $convertion_currency;
            if(isset($convertion_currency[$id_currency_need]) && $convertion_currency[$id_currency_need]>0)
                return $sum*$convertion_currency[$id_currency_have]/$convertion_currency[$id_currency_need];
            else
                return FALSE;
        }
        else
            return 0;
    }

    function get_id_group_originator($title_group_originators)
    {
        global $SQL;
        $query = 'select id_group_originator from sms_group_originators where title_group_originators="' . $title_group_originators . '"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
            $id_group_originator = $SQL->result($result, 0, 'id_group_originator');
        else
        {
            $query = 'insert into sms_group_originators set title_group_originators="' . $title_group_originators . '"';
            $SQL->query($query);
            $id_group_originator = $SQL->insert_id();
        }
        return $id_group_originator;
    }

    function get_id_group_originator_scheme_traffic($title_group_originators)
    {
        global $SQL;
        $query = 'select id_group_originator from sms_group_originator where title_group_originators="' . $title_group_originators . '"';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
            $id_group_originator = $SQL->result($result, 0, 'id_group_originator');
        else
        {
            $query = 'insert into sms_group_originator set title_group_originators="' . $title_group_originators . '"';
            $SQL->query($query);
            $id_group_originator = $SQL->insert_id();
        }
        return $id_group_originator;
    }

    function set_replace_originators($id_aggregating)
    {
        global $SQL, $global_replace_originators;
        $global_replace_originators = array();
        $query = 'select MCC, MNC, id_group_originator from sms_aggregating_originator where id_aggregating="' . $id_aggregating . '"';
        $result = $SQL->query($query);
        while($res = $SQL->fetch_assoc($result))
        {
            $query = 'select originator from sms_originators where id_group_originator="' . $res['id_group_originator'] . '"';
            $result_or = $SQL->unbuffered_query($query);
            while($res_or = $SQL->fetch_assoc($result_or))
            {
                $global_replace_originators[$res['MCC']][$res['MNC']]['originator'][] = $res_or['originator'];
            }
            $global_replace_originators[$res['MCC']][$res['MNC']]['count'] = count($global_replace_originators[$res['MCC']][$res['MNC']]['originator'])-1;
        }
    }

    function get_replace_originators($MCC, $MNC)
    {
        global $global_replace_originators;
        if($global_replace_originators[$MCC][$MNC]['count'])
            return $global_replace_originators[$MCC][$MNC]['originator'][rand(0, $global_replace_originators[$MCC][$MNC]['count'])];
        elseif($global_replace_originators[$MCC][0]['count'])
            return $global_replace_originators[$MCC][0]['originator'][rand(0, $global_replace_originators[$MCC][0]['count'])];
        else
            return FALSE;
    }

    function get_param_state_id($id_message, $id_aggregating)
    {
        global $SQL;
        $query = 'select id_sms from sms_state_final where id_part_operator="' . $id_message . '" and id_aggregating="' . $id_aggregating . '" and part_no="1"';
        $result = $SQL->query($query);
        if($res = $SQL->fetch_assoc($result))
            return $res;
        return FALSE;
    }
    
    function js_quotes_gpc($string)
    {
        $string = str_replace(chr(92), chr(92).chr(92), $string);
        $string = str_replace('"', '&quot;', $string);
        $string = str_replace('<', '&lt;', $string);
        $string = str_replace('>', '&gt;', $string);
        return $string;
    }
    function send_SMSC_sms_admin($message, $type = 'privilege')
    {
        global $SQL, $param_site;
 
        $choice = 'select * from admin_users';
        $result = $SQL->query($choice);
        $num = $SQL->num_rows($result);
        $phones = array();
        for($i=0; $i<$num; $i++)
        {
            $phone = $SQL->result($result, $i, 'phone');
            $send = $SQL->result($result, $i, $type);
            if($send&&$phone){
                array_push($phones, $phone);
            }
                
        }
        $url = 'http://smsc.ru/sys/send.php?login=SMSC_Direct_All&psw=Inkor&charset=utf-8&phones='. implode(',', $phones).'&mes='.$message;
        return send_get($url);
    }
	function save_event($data, $type = 'pay', $login){
		global $SQL;
		if($type == 'pay'){
			$query = "SELECT * FROM `events2crm` WHERE `u_login` = '" . $login . "' AND `type` = 'reg'";
			$result = $SQL->query($query);
			$num = $SQL->num_rows($result);
			if(!$num){
				$query = "SELECT `login`, `name`, `e_mail`, `phone` FROM `sms_send_users` WHERE `id_user` = '" . $data['id_client'] . "'";
				$result_reg = $SQL->query($query);
				$login = $SQL->result($result_reg, 0, 'login');
				$name = $SQL->result($result_reg, 0, 'name');
				$e_mail = $SQL->result($result_reg, 0, 'e_mail');
				$phone = $SQL->result($result_reg, 0, 'phone');
					
				if(empty($name)){
					$name = $e_mail;
				}
				if(empty($name)){
					$name = $login;
				}
				
				$data_reg = array();
				$data_reg['id_client'] = $data['id_client'];
				$data_reg['login'] = $login;
				$data_reg['phone'] = $phone;
				$data_reg['e_mail'] = $e_mail;
				$data_reg['name'] = $name;
				$query = "INSERT INTO `events2crm` SET `time` = '" . date('Y.m.d h:i:s', time()) . "', `type` = 'reg', `data` = '" . serialize($data_reg) . "', `u_login` = '" . $login . "', `server` = 'my5'";
				$result = $SQL->query($query);
			}			
		}
		$query = "INSERT INTO `events2crm` SET `time` = '" . date('Y.m.d h:i:s', time()) . "', `type` = '" . $type . "', `data` = '" . serialize($data) . "', `u_login` = '" . $login . "', `server` = 'my5'";
		$result = $SQL->query($query);
	}
        
    function get_param_aggregating_hlr($id_aggregating)
    {
        global $SQL;
        $query = 'select sql_cache *  from sms_aggregating_HLR where id_aggregating="' . $id_aggregating . '"';
        $result = $SQL->query($query);
        if($res = $SQL->fetch_assoc($result))
            return $res;
        return FALSE;
    }
    
    function check_smpp_sender_hlr($smpp)
    {
        global $SQL;
        $query = 'select sql_cache num_send, reconnect, state_time, time_zone, bind from sms_aggregating_HLR where type="SMPP" and ip!="" and port!="" and num_send>0 and id_aggregating="' . $smpp->id_aggregating . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $smpp->state_time = $SQL->result($result, 0, 'state_time');
            $smpp->time_zone = $SQL->result($result, 0, 'time_zone');
            $check_smpp['num_send'] = $SQL->result($result, 0, 'num_send');
            $reconnect = $SQL->result($result, 0, 'reconnect');
            if($reconnect == '2')
            {
                $smpp->type = $SQL->result($result, 0, 'bind');
                $query = 'update sms_aggregating_hlr set reconnect="1" where id_aggregating="' . $smpp->id_aggregating . '"';
                $SQL->query($query);
                $smpp->_socket = FALSE;
                $check_smpp['start_smpp'] = $smpp->StartSmpp();
            }
            return $check_smpp;
        }
        else
        {
            $smpp->End();
            $smpp->set_data();
            exit();
        }
    }
    
    function get_param_state_hlr_id($id_message, $id_aggregating)
    {
        global $SQL;
        $query_hlr = 'select * from sms_turn_HLR where id_part_operator="' . $id_message . '" and id_aggregating_HLR="' . $id_aggregating . '"';
        $result_hlr = $SQL->query($query_hlr);
        if($res_hlr = $SQL->fetch_assoc($result_hlr)){
            $query = 'select * from sms_turn where 	id_turn="' . $SQL->result($result_hlr, 0, 'id_turn') . '"';
            $result = $SQL->query($query);
            if($res = $SQL->fetch_assoc($result))    
                return $res;
        }
        return FALSE;
    }
    
    function set_sms_turn_hlr_delive($id_message, $id_aggregating){
        global $SQL, $query_log;
        $query = 'select * from sms_turn_HLR where id_part_operator="' . (int)$id_message . '" and id_aggregating_HLR="' . (int)$id_aggregating . '"';
        
        $result = $SQL->query($query);
        if($res = $SQL->fetch_assoc($result)){
            if($res['id_turn']>0){
                $query = 'UPDATE `sms_turn` SET `status_hlr`=0 WHERE `id_turn`="'.$res['id_turn'].'"';
                $SQL->query($query);
                $affected_rows = $SQL->affected_rows();
                
            }else{
                $query = 'UPDATE `sms_turn_time` SET `status_hlr`=0 WHERE `id_turn`="'.$res['id_turn'].'"';
                $SQL->query($query);
                $affected_rows = $SQL->affected_rows();
            }
            
            $query_log = $query;
            $query = 'delete from sms_turn_HLR where id_turn="'.$res['id_turn'].'"';
            $SQL->query($query);
            return $affected_rows;
        }
        
        return false;
    }
    
    function set_final_state_hlr($id_turn)
    {
        global $SQL;
        $query = 'select * from sms_turn_HLR where id_turn="' . $id_turn . '"';
        
        $result = $SQL->query($query);
        if($res = $SQL->fetch_assoc($result)){
            if($id_turn<0){
                    set_convertion_currency();
                    $date = date('Y-m-d H:i:s');
                    $query = 'insert into sms_send_trafic (id_sms, id_user, id_package_user, id_partner, id_package_partner, MCC, MNC, MNC_user, MNC_partner, time_put_turn, originator, phone, type_sms, text_sms, num_parts, id_name_delivery, validity_period, time) (select -1*cast(id_turn as signed), id_user, id_package_user, id_partner, id_package_partner,  MCC,MNC , MNC_user, MNC_partner, time_put_turn, originator, phone, type_sms, text_sms, count_sms, id_name_delivery, validity_period,"'.$date.'" from sms_turn_time where id_turn="'.$id_turn.'")';
                    $SQL->query($query);
                    if($SQL->affected_rows() > 0)
                    {
                        $query = 'SELECT * from sms_turn_time where id_turn="'.$id_turn.'"';
                        $result = $SQL->query($query);
                        if($res = $SQL->fetch_assoc($result))
                        {
                            $query = 'delete from sms_turn_time where id_turn="'.$id_turn.'" and id_control_text!=0';
                            $SQL->query($query);
                            $query = 'UPDATE sms_state_final SET state="not_deliver", final="1" where id_sms="-'.$id_turn.'"';
                            $SQL->query($query);
                            $date = substr($date, 0, 13) . ':00:00';
                            $param = get_param_id_user($res['id_user']);
                            update_sms_stat($res['id_user'],$res['id_partner'],$param['id_manager'],$res['id_currency'], 0,$res['MCC'],$res['MNC'],$res['id_name_delivery'],$date,$res['originator'],'not_deliver',0,$res['count_sms'],$res['sell_we'],$res['buy_we'],$res['sell_partner'],$res['buy_partner']);
                        }
                    }
                    $query = 'delete from sms_turn_HLR where id_turn="'.$id_turn.'"';
                    $SQL->query($query);
                    return 'final sms_turn_time';
                }else{
                    set_convertion_currency();
                    $date = date('Y-m-d H:i:s');
                    $query = 'insert into sms_send_trafic (id_sms, id_user, id_package_user, id_partner, id_package_partner, MCC, MNC, MNC_user, MNC_partner, time_put_turn, originator, phone, type_sms, text_sms, num_parts, id_name_delivery, validity_period, time) (select cast(id_turn as signed), id_user, id_package_user, id_partner, id_package_partner,  MCC,MNC , MNC_user, MNC_partner, time_put_turn, originator, phone, type_sms, text_sms, count_sms, id_name_delivery, validity_period,"'.$date.'" from sms_turn where id_turn="'.$id_turn.'")';
                    $SQL->query($query);
                    if($SQL->affected_rows() > 0)
                    {
                        $query = 'SELECT * from sms_turn where id_turn="'.$id_turn.'"';
                        $result = $SQL->query($query);
                        if($res = $SQL->fetch_assoc($result))
                        {
                            $validity_period = date('Y-m-d H:i:s',time()+86400);
                            if(strtotime($res['validity_period'])<strtotime($validity_period) && strtotime($res['validity_period'])>strtotime($date)){
                                $validity_period = $res['validity_period'];
                            }
                            $param = get_param_id_user($res['id_user']);
                            $date = substr($date, 0, 13) . ':00:00';
                            $query = 'UPDATE sms_turn_HLR SET status="2", datetime_final="' . $validity_period . '" where id_turn="'.$id_turn.'"';
                            $SQL->query($query);
                            update_sms_stat($res['id_user'],$res['id_partner'],$param['id_manager'],$res['id_currency'], 0,$res['MCC'],$res['MNC'],$res['id_name_delivery'],$date,$res['originator'],'partly_deliver',0,$res['count_sms'],$res['sell_we'],$res['buy_we'],$res['sell_partner'],$res['buy_partner']);
                        
                            $query = 'delete from sms_turn where id_turn="'.$id_turn.'"';
                            $SQL->query($query);
                        }
                    }
                    return $query;
                }
        }
    }
        
    function get_id_aggregating_hlr($id_user, $id_partner, $MCC,$MNC, $type_sms,$originator, $count_sms,$turn_id_sms=FALSE)
    {
        global $SQL;
        $id_aggregating_result = 0;
//        if($id_partner)
//            return 0;
        $query = 'select sql_cache id_group_aggregating from sms_scheme_traffic_HLR where id_client="' . $id_user . '" and MCC="'.$MCC.'" and (MNC="'.$MNC.'" or MNC=0) and (BINARY "'.addslashes($originator).'" like originator or originator = "") and type_sms="'.$type_sms.'" order by priority asc, MNC desc';
        $result_aggregating = $SQL->query($query);
        $num_aggregating = $SQL->num_rows($result_aggregating);
        for($i=0; $i<$num_aggregating; $i++)
        {
            $id_group_aggregating = $SQL->result($result_aggregating, $i, 'id_group_aggregating');
            $query = 'select sql_cache sd.id_aggregating, sd.percent, sd.main from sms_distribution_HLR as sd, sms_aggregating_HLR as sa where sd.id_aggregating=sa.id_aggregating and sd.id_group_aggregating="' . $id_group_aggregating . '" and switch="on" and sa.works="1" and sa.cope="1" and sa.num_sms>=' . $count_sms;
            $result = $SQL->query($query);
            $num = $SQL->num_rows($result);
            $percent = 0;
            $percent_main=0;
            unset($aggregating);
            unset($aggregating_main);
            for($j=0; $j<$num; $j++)
            {
                $id_aggregating = $SQL->result($result, $j, 'sd.id_aggregating');
                if($used[$id_aggregating])
                   continue;
                $main=$SQL->result($result, $j, 'sd.main');
                if($main)
                {
                   $percent_main+= $SQL->result($result, $j, 'sd.percent');
                   $aggregating_main[$j]['id_aggregating'] = $id_aggregating;
                   $aggregating_main[$j]['percent'] = $percent_main;
                }
                else
                {
                   $percent += $SQL->result($result, $j, 'sd.percent');
                   $aggregating[$j]['id_aggregating'] = $id_aggregating;
                   $aggregating[$j]['percent'] = $percent;
                }
            }
            if(is_array($aggregating_main))
            {
                $rand = rand(1, $percent_main);
                foreach($aggregating_main as $am_value)
                {
                    if($rand <= $am_value['percent'])
                    {
                        $id_aggregating_result = $am_value['id_aggregating'];
                        break 2;
                    }
                }
            }
            else
            {
                if(is_array($aggregating))
                  {
                       $rand = rand(1, $percent);
                         foreach($aggregating as $agg_value)
                         {
                             if($rand <= $agg_value['percent'])
                             {
                                 $id_aggregating_result = $agg_value['id_aggregating'];
                                 break 2;
                             }
                         }
                  }
            }
        }

        if($id_aggregating_result)
        {
            $query = 'update sms_aggregating_HLR set num_sms=num_sms-' . $count_sms . ' where id_aggregating="' . $id_aggregating_result . '"';
            $SQL->query_async($query);
        }
        return $id_aggregating_result;
    }
        
    function send_mail_incoredevelopment($message='',$title=''){
        $new_message = '<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=Windows-1251" />
<title>Incore Development - '.$title.'.</title>
</head>
<body bgcolor="#FFFFFF"><br>
<table align="center" width="600" border="0" cellpadding="0" cellspacing="0">
    <tr height="70" bgcolor="#00b9c0">
        <td bgcolor="#00b9c0" width="200" align="left"><a href="http://www.incoredevelopment.com/" target="_blank" style="text-decoration: none;">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<img src="http://s017.radikal.ru/i435/1507/1f/d491d797022b.png" alt="Incore Development" width="163" height="52" border="0" /></a></td>
        <td width="200" align="right"><span style="font-family: Arial, Helvetica, sans-serif; font-size: 16px; color: #ffffff;font-weight: bold;">8 (800)</span><span style="font-family: Arial, Helvetica, sans-serif; font-size: 24px; color: #ffffff;font-weight: bold; padding-right:30px">&nbsp;550-17-89&nbsp;</span><br><span style="font-family: Arial, Helvetica, sans-serif; font-size: 12px; color: #ffffff;font-weight: bold; padding-right:30px">(бесплатно по всей России)&nbsp;&nbsp;&nbsp;</span></td>        
    </tr>
</table>
<table align="center" width="600" border="0" cellpadding="0" cellspacing="0">
	<tr>
            <td colspan="4" height="30" bgcolor="#f8f8f8">&nbsp;</td>
	</tr>
	<tr>
            <td width="28" bgcolor="#f8f8f8">&nbsp;</td>
            <td valign="top" colspan="2"  bgcolor="#f8f8f8">';
                if($title!='')
                    $new_message .= '<p align="center"><span style="font-family: Arial, Helvetica, sans-serif; font-size: 30px; color: #292929;">'.$title.'</br></span></p>';
                $new_message .= '<div align="left" style="font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #292929;">'.$message.'</div>';
                $new_message .= '<p align="left"><span style="font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #292929;">С уважением, <strong>Инкор Девелопмент</strong></p>
            </td>';
        $new_message .= '<td width="30" bgcolor="#f8f8f8">&nbsp;</td>
	</tr>
	<tr>
            <td colspan="4" height="30" bgcolor="#f8f8f8">&nbsp;</td>
	</tr>
</table>
<table height="100" align="center" width="600" border="0" cellpadding="0" cellspacing="0" bgcolor="#353535">
    <tr>
        <td height="38" width="165" style="padding-top:25px;padding-left:50px;padding-bottom:0px;padding-right:0;color:#ffffff;font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #ffffff;text-decoration: underline;text-align:left"><a style="color:#ffffff; text-decoration: underline;" href="http://www.incoredevelopment.com/rates" title="Тарифы" target="_blank">Тарифы</a></td>        
        <td height="38" width="125" style="padding-top:25px;padding-left:0px;padding-bottom:0px;padding-right:0;color:#ffffff;font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #ffffff;text-decoration: underline;text-align:left"><a style="color:#ffffff; text-decoration: underline;" href="https://my5.incore1.ru/ru/cabinet.html" title="Вход" target="_blank">Вход</a></td>
        <td width="38" rowspan="4">&nbsp;</td>
        <td bgcolor="#353535" height="38" width="249" style="padding-top:25px;padding-left:0px;padding-bottom:0px;padding-right:0;color:#ffffff;font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #ffffff;text-decoration: none;text-align:left">Адрес:</td>
        <td width="23" rowspan="4">&nbsp;</td>
    </tr>
    <tr>
        <td height="38" style="padding-top:0px;padding-left:50px;padding-bottom:0px;padding-right:0;color:#ffffff;font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #ffffff;text-decoration: underline;text-align:left"><a style="color:#ffffff; text-decoration: underline;" href="http://www.incoredevelopment.com/#general" title="Услуги" target="_blank">Услуги</a></td>
        <td style="padding-top:1px;padding-left:0px;padding-bottom:0px;padding-right:0;color:#ffffff;font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #ffffff;text-decoration: underline;text-align:left"><a style="color:#ffffff; text-decoration: underline;" href="https://my5.incore1.ru/ru/reg.html" title="Регистрация" target="_blank">Регистрация</a></td>
        <td><span style="color:#ffffff;font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #ffffff;text-decoration: none;text-align:left">г. Москва, ул. Складочная д. 1, стр.18, оф.218</span></td>
    </tr>
    <tr>
        <td height="38" style="padding-top:0px;padding-left:50px;padding-bottom:0px;padding-right:0;color:#ffffff;font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #ffffff;text-decoration: underline;text-align:left"><a style="color:#ffffff; text-decoration: underline;" href="http://www.incoredevelopment.com/#NaN" title="Решения" target="_blank">Решения</a></td>
        <td style="padding-top:0px;padding-left:0px;padding-bottom:0px;padding-right:0;color:#ffffff;font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #ffffff;text-decoration: underline;text-align:left"><a style="color:#ffffff; text-decoration: underline;" href="http://www.incoredevelopment.com/news/" title="Новости компании" target="_blank">Новости</a></td>
        <td><a style="font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #00b9c0;text-decoration: none;" href="mailto:info@incoredevelopment.com" target="_blank">info@incoredevelopment.com</a></td>       
    </tr>
    <tr>
        <td height="38" style="padding-top:0px;padding-left:50px;padding-bottom:15px;padding-right:0;color:#ffffff;font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #ffffff;text-decoration: underline;text-align:left"><a style="color:#ffffff; text-decoration: underline;" href="http://www.incoredevelopment.com/#man" title="Партнерская программа" target="_blank">Партнерская программа</a></td>
        <td style="padding-top:0px;padding-left:0px;padding-bottom:10px;padding-right:0;color:#ffffff;font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #ffffff;text-decoration: underline;text-align:left"><a style="color:#ffffff; text-decoration: underline;" href="http://www.incoredevelopment.com/contacts" title="Контакты" target="_blank">Контакты</a></td>
        <td style="padding-top:0px;padding-left:0px;padding-bottom:10px;padding-right:0;"><span style="font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #00b9c0;text-decoration: underline;"><a style="color:#00b9c0; text-decoration: underline;" href="http://www.incoredevelopment.com/" target="_blank">www.incoredevelopment.com</a></span></p></td>
    </tr>
</table>
<table align="center" width="600" border="0" cellpadding="0" cellspacing="0">
    <tr>
        <td height="40" align="center" bgcolor="#292929"; style="font-family: Arial, Helvetica, sans-serif; font-size: 12px; color: #ffffff;">© 2016 InCore Development. Все права защищены.</td>
    </tr>    
</table>
</body>
</html>';
        return $new_message;
    }
    
    function delete_test_originator($id_user=0,$originator='',$id_partner='0'){
        global $SQL;
        if($id_user){
            if(!$originator){
                $param_site = get_param_site($id_partner);
                $originator = $param_site["SENDER_TEST_SMS"];
            }
            if($originator){
                $query = 'delete from sms_originator where id_user="' . $id_user . '" BINARY and originator="' . $originator . '"';
                $SQL->query($query);
            }
        }
    } 
    
    function get_client_by_inn($inn){
        global $SQL;
        $return = array();
        if($inn){
            $query = 'SELECT `id_user` FROM `sms_send_users` WHERE `inn`="' . $inn . '" LIMIT 1';
            $result = $SQL->query($query);
            $num = $SQL->num_rows($result);
            for($i=0; $i<$num; $i++)
            {
                $id_user = $SQL->result($result, $i, 'id_user');
                $return['type']='user';
                $return['id']=$id_user;
            }
            $query = 'SELECT `id_partner` FROM `sms_partners` WHERE `inn`="' . $inn . '" LIMIT 1';
            $result = $SQL->query($query);
            $num = $SQL->num_rows($result);
            for($i=0; $i<$num; $i++)
            {
                $id_partner = $SQL->result($result, $i, 'id_partner');
                $return['type']='partner';
                $return['id']=$id_partner;
            }
            return $return;
        }else{
            return FALSE;
        }
    }
     function get_login_by_id_partner($id_partner){
        global $SQL;
        $query = 'SELECT `login` FROM `sms_partners` WHERE `id_partner`="' . $id_partner . '" LIMIT 1';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
        {
            return $SQL->result($result, 0, 'login');
        }
    }
    function get_name_by_id_manager($id_manager){
        global $SQL;
        $query = 'SELECT `name` FROM `sms_manager` WHERE `id_manager`="' . $id_manager . '" LIMIT 1';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($num)
        {
            return $SQL->result($result, 0, 'name');
        }
    }
    function get_options_month($id_month)
    {
        $option = '';
        for($i=1; $i<=12; $i++)
        {
            if($i == $id_month)
                $selected = ' selected';
            else
                $selected = '';
            $option .= '<option value="' . $i . '"' . $selected . '>' . get_short_text_month($i) . '</option>';
        }
        return $option;
    }
    /**
     * Получение и сравнение доступных способов оплаты от робокассы
     * @global type $pay_menu
     * @param type $login
     * @return type
     */
    function robox_availability_get_pay_metod($login){
        global $pay_menu;
        $url = 'https://auth.robokassa.ru/Merchant/WebService/Service.asmx/GetCurrencies?MerchantLogin='.$login.'&Language=ru';
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_CRLF, true);
        curl_setopt($ch, CURLOPT_URL, $url);
        $result_curl = curl_exec($ch);
        $res = $result_curl;
        curl_close($ch);
        $pay_metod_availability = new SimpleXMLElement($res);
        $pay_metod_robox = array();
        if ($pay_metod_availability->Result == true && $pay_metod_availability->Result->Code == 0) {
            
            foreach ($pay_metod_availability->Groups->Group as $pay_metod) {
                $lvl2 = array();
                foreach ($pay_metod->Items->Currency as $val){
                    $lvl2[(string) $val['Alias']] = (string) $val['Label'];
                }
                $pay_metod_robox[(string) $pay_metod['Code']]=array('name'=>(string) $pay_metod['Description'], 'type_payments'=>$lvl2 );
                
                
            }
        
            $result =array();
            foreach ($pay_menu['robo'] as $key=>$val){
                $result[$key]['name'] = $pay_menu['robo'][$key]['name'];
                $result[$key]['type_payments'] = array_intersect_key($val['type_payments'], $pay_metod_robox[$key]['type_payments']);
            }
            $pay_menu['robo'] = $result;
            return $pay_menu;
        }else{
            $pay_menu['robo'] = array();
            return false;
        }
        
    }
    /**
     * Получение и сравнение доступных способов оплаты от simplepay.pro
     * @global type $pay_menu
     * @param type $login
     * @return type
     */
    function simplepay_availability_get_pay_metod($sp_outlet_id,$secret_key){
        global $pay_menu;
        $sp_amount = '1';
        $sp_salt = rand(50,999);
        $sig = md5('ps_list;'.$sp_amount.';'.$sp_outlet_id.';'.$sp_salt.';'.$secret_key);
        $url = "http://api.simplepay.pro/sp/ps_list?sp_amount={$sp_amount}&sp_outlet_id={$sp_outlet_id}&sp_salt={$sp_salt}&sp_sig=".$sig;
  
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_CRLF, true);
        curl_setopt($ch, CURLOPT_URL, $url);
        $result = curl_exec($ch);
        $res = $result;
        curl_close($ch);
        $pay_metod_availability = new SimpleXMLElement($res);
        $pay_metod_simple = array();
        
        if ((string)$pay_metod_availability->sp_status == 'ok' ) {
            foreach ($pay_metod_availability->payment_systems->sp_payment_system as $pay_metod) {
            $pay_metod_simple[(string)$pay_metod->sp_alias] = $pay_metod->sp_name;
            }
           
            $result =array();
            foreach ($pay_menu['robo'] as $key=>$val){
                $result[$key]['name'] = $pay_menu['robo'][$key]['name'];
                $result[$key]['type_payments'] = array_intersect_key($val['type_payments'], $pay_metod_simple);
            }
            $pay_menu['robo'] = $result;
            return $pay_menu;
        }else{
            $pay_menu['robo'] = array();
            return false;
        }
        return $pay_menu;
    }
    
    function check_user_ip_xml($id_user=0){
        global $SQL;
        $ip =  $_SERVER["REMOTE_ADDR"];
        $query = 'select * from sms_ip_xml where ip="' . $ip . '" and id_user="' . $id_user . '"';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
            return TRUE;
        else
        {
            $query = 'select * from sms_ip_xml where id_user="' . $id_user . '"';
            $result = $SQL->query($query);
            if($SQL->num_rows($result)){
                $query = 'select * from sms_ip_xml where id_user="' . $id_user . '" and (ip LIKE ("%/%") or ip LIKE ("%*%") or ip LIKE ("%-%"))';
                $result = $SQL->query($query);
                $num = $SQL->num_rows($result);
                if($num>0){
                    for($i=0; $i<$num; $i++)
                    {
                        $range = $SQL->result($result, $i, 'ip');
                        if(ip_in_range($ip, $range))
                            return TRUE;

                    }
                    return FALSE;
                }
                else{
                    return FALSE;
                }
            } else
                return TRUE;
        }
    }
    
    function is_valid_inn($inn)
    {
        $inn = (string) $inn;
        $len = strlen($inn);

        if ( $len === 10 )
        {
            return $inn[9] === (string) (((
                2*$inn[0] + 4*$inn[1] + 10*$inn[2] +
                3*$inn[3] + 5*$inn[4] +  9*$inn[5] +
                4*$inn[6] + 6*$inn[7] +  8*$inn[8]
            ) % 11) % 10);
        }
        elseif ( $len === 12 )
        {
            $num10 = (string) (((
                 7*$inn[0] + 2*$inn[1] + 4*$inn[2] +
                10*$inn[3] + 3*$inn[4] + 5*$inn[5] +
                 9*$inn[6] + 4*$inn[7] + 6*$inn[8] +
                 8*$inn[9]
            ) % 11) % 10);

            $num11 = (string) (((
                3*$inn[0] +  7*$inn[1] + 2*$inn[2] +
                4*$inn[3] + 10*$inn[4] + 3*$inn[5] +
                5*$inn[6] +  9*$inn[7] + 4*$inn[8] +
                6*$inn[9] +  8*$inn[10]
            ) % 11) % 10);

            return $inn[11] === $num11 && $inn[10] === $num10;
        }

        return false;
    }
    
    function add_originators_xls($sheets, $data, $id_background, $operators=array(), $id_user, $id_partner, $id_manager, $inn_client)
    {
        global $SQL, $global_languages;
        $sheet_count = 0;
        $id_partner = $id_manager = 0;
        $query = 'SELECT `id_partner`,`id_manager` FROM `sms_send_users` WHERE `id_user`="' . $id_user . '" LIMIT 1';
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        for($i=0; $i<$num; $i++){
            $id_partner = $SQL->result($result, $i, 'id_partner');
            $id_manager = $SQL->result($result, $i, 'id_manager');
        }
        for($sheet=0;$sheet<count($data->sheets);$sheet++)
        {
            if(isset($sheets[$sheet_count]))
            {
                $rest_array[$sheet_count] = $data->sheets[$sheet]["numRows"];
            }
            if($data->sheets[$sheet]['numCols'])
                $sheet_count++;
        }
        $originator_insert = array();
        $count_load = 0;
        $num_load = 0;
        $sheet_count = 0;
        $operators_all = get_operators_all();
        
        for($sheet=0;$sheet<count($data->sheets);$sheet++)
        {
            if(isset($sheets[$sheet_count]))
            {
            $count_rows = $data->sheets[$sheet]["numRows"];
            for($row=1;$row<=$data->sheets[$sheet]["numRows"]&&($row<=$max_rows||$max_rows==0);$row++)
            {
                $phones = array();
                for($col=1;$col<=$data->sheets[$sheet]["numCols"]&&($col<=$max_cols||$max_cols==0);$col++)
                {
                    if($data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"] >=1 && $data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"] >=1)
                    {
                        $this_cell_colspan = ' colspan=' . $data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];
                        $this_cell_rowspan = ' rowspan=' . $data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"];
                        for($i=1;$i<$data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];$i++)
                        {
                            $data->sheets[$sheet]["cellsInfo"][$row][$col+$i]["dontprint"]=1;
                        }
                        for($i=1;$i<$data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"];$i++)
                        {
                            for($j=0;$j<$data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];$j++)
                            {
                                $data->sheets[$sheet]["cellsInfo"][$row+$i][$col+$j]["dontprint"]=1;
                            }
                        }
                    }
                    elseif($data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"] >=1)
                    {
                        $this_cell_colspan = ' colspan=' . $data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];
                        $this_cell_rowspan = '';
                        for($i=1;$i<$data->sheets[$sheet]["cellsInfo"][$row][$col]["colspan"];$i++)
                        {
                            $data->sheets[$sheet]["cellsInfo"][$row][$col+$i]["dontprint"]=1;
                        }
                    }
                    elseif($data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"] >=1)
                    {
                        $this_cell_colspan = '';
                        $this_cell_rowspan = ' rowspan=' . $data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"];
                        for($i=1;$i<$data->sheets[$sheet]["cellsInfo"][$row][$col]["rowspan"];$i++)
                        {
                            $data->sheets[$sheet]["cellsInfo"][$row+$i][$col]["dontprint"]=1;
                        }
                    }
                    else
                    {
                        $this_cell_colspan = '';
                        $this_cell_rowspan = '';
                    }

                    if(!($data->sheets[$sheet]["cellsInfo"][$row][$col]["dontprint"]))
                    {
                        $phones[$col-1] = nl2br(htmlentities($data->sheets[$sheet]["cells"][$row][$col], ENT_COMPAT, 'UTF-8'));
                    }
                }

                if(is_array($sheets[$sheet_count]))
                {
                    $i = 0;
                    $inn = '';
                    $legal_entity = '';
                    $originator = '';
                    $comments = '';
                    foreach($sheets[$sheet_count] as $key => $val)
                    {
                        if($val)
                            ${$val} = check_xls_value($phones[$key]);
                        $i++;
                    }
                    if(count($operators)>0){
                        foreach($operators as $operator_one){
                            $MCC = $operators_all[$operator_one]["MCC"];
                            $MNC = $operators_all[$operator_one]["MNC"];
                            $type_originator = $operators_all[$operator_one]["type_originator"];
                            if(is_valid_inn($inn)){
                                if($legal_entity){
                                    $query = "SELECT `id_originator` FROM `bill_register_originator` WHERE `originator`='{$originator}' AND `inn`='{$inn}' AND `id_user`='{$id_user}' AND `MCC`='{$MCC}' AND `MNC`='{$MNC}' AND `type_originator`='{$type_originator}' LIMIT 1";
                                    $result = $SQL->query($query);
                                    $num = $SQL->num_rows($result);
                                    if($num>0){
                                        error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $global_languages["SENDER_ALLREADY_EXISTS"]);
                                    } else {
                                        if(check_originator($originator)){
                                            $count_load++;
                                            $num_load += add_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $id_partner, $id_manager, $inn_client);
                                        } else {
                                            error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $global_languages["SENDER_CAN_ONLY_CONSIST"]);
                                        }
                                    } 
                                } else {
                                    error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $global_languages["ERROR_LEGAL_ENTITY"]);
                                }
                            } else {
                                error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $global_languages["ERROR_INN"]);
                            }
                        }
                    }
                }
                $rest_array[$sheet_count]--;
                if($count_load%25 == 0)
                    set_num_sms($id_background, 1, $rest_array, $num_load);
            }
            }
            if($data->sheets[$sheet]['numCols'])
                $sheet_count++;
        }
        set_num_sms($id_background, 1, $rest_array, $num_load);
        return $num_load;
    }
    
    function add_originators_xlsx($sheets, $file, $id_background, $operators=array(), $id_user, $id_partner, $id_manager, $inn_client)
    {
        global $SQL, $global_languages;
        if(file_exists($file))
        {
            $xlsx = new SimpleXLSX($file);

            $sheet_count = 0;
            $num_sheets = $xlsx->sheetsCount();
            for($sheet=0;$sheet<$num_sheets;$sheet++)
            {
                list($num_cols, $num_rows) = $xlsx->dimension($sheet+1);
                if(isset($sheets[$sheet_count]))
                    $rest_array[$sheet_count] = $num_rows;
                if($num_cols && $num_rows)
                    $sheet_count++;
            }
            $phones_insert = array();
            
            $count_load = 0;
            $num_load = 0;
            $sheet_count = 0;
            $operators_all = get_operators_all();
            $num_sheets = $xlsx->sheetsCount();
            for($sheet=0;$sheet<$num_sheets;$sheet++)
            {
                list($num_cols, $num_rows) = $xlsx->dimension($sheet+1);
                if(isset($sheets[$sheet_count]))
                {
                $rows = $xlsx->rows($sheet+1);
                for($j=0; $j<$num_rows; $j++)
                {
                    $phones = array();
                    for($i=0; $i<$num_cols; $i++)
                    {
                        $phones[$i] = nl2br(htmlentities($rows[$j][$i], ENT_COMPAT, 'UTF-8'));
                    }
                    if(is_array($sheets[$sheet_count]))
                    {
                        $i = 0;
                        $inn = '';
                        $legal_entity = '';
                        $originator = '';
                        $comments = '';
                        foreach($sheets[$sheet_count] as $key => $val)
                        {
                            if($val && $phones[$key])
                            {
                                ${$val} = check_xls_value($phones[$key]);
                            }
                            $i++;
                        }
                        if(count($operators)>0){
                            foreach($operators as $operator_one){
                                $MCC = $operators_all[$operator_one]["MCC"];
                                $MNC = $operators_all[$operator_one]["MNC"];
                                $type_originator = $operators_all[$operator_one]["type_originator"];
                                if(is_valid_inn($inn)){
                                    if($legal_entity){
                                        $query = "SELECT `id_originator` FROM `bill_register_originator` WHERE `originator`='{$originator}' AND `inn`='{$inn}' AND `id_user`='{$id_user}' AND `MCC`='{$MCC}' AND `MNC`='{$MNC}' AND `type_originator`='{$type_originator}' LIMIT 1";
                                        $result = $SQL->query($query);
                                        $num = $SQL->num_rows($result);
                                        if($num>0){
                                            error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $global_languages["SENDER_ALLREADY_EXISTS"]);
                                        } else {
                                            if(check_originator($originator)){
                                                $count_load++;
                                                $num_load += add_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $id_partner, $id_manager, $inn_client);
                                            } else {
                                                error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $global_languages["SENDER_CAN_ONLY_CONSIST"]);
                                            }
                                        } 
                                    } else {
                                        error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $global_languages["ERROR_LEGAL_ENTITY"]);
                                    }
                                } else {
                                    error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $global_languages["ERROR_INN"]);
                                }
                            }
                        }
                    }
                    $rest_array[$sheet_count]--;
                    if($count_load%25 == 0)
                        set_num_sms($id_background, 1, $rest_array, $num_load);
                }
                }
                if($num_cols && $num_rows)
                    $sheet_count++;
            }
        }
        set_num_sms($id_background, 1, $rest_array, $num_load);
        return $num_load;
    }
    
    function add_originators_csv($sheets, $file, $id_background, $operators=array(), $id_user, $id_partner, $id_manager, $inn_client)
    {
        global $SQL, $global_languages;
        if(file_exists($file))
        {
        $phones_insert = array();
        $count_load = 0;
        $num_load = 0;
        $fp = fopen($file, 'r');
        $operators_all = get_operators_all();
        $rest = 0;
        while(($string = fgets($fp, 3072000)) !== FALSE && isset($sheets[0]))
        {
            $rest++;
        }
        rewind($fp);

        while(($string = fgets($fp, 3072000)) !== FALSE && isset($sheets[0]))
        {
            $phones = array();
            $string = iconv('Windows-1251', 'UTF-8', $string);
            $data = explode(';', $string);
            $num = count($data);
            for($c=0; $c < $num; $c++)
            {
                if(substr($data[$c], 0, 1) == '"')
                    $data[$c] = substr($data[$c], 1);
                if(substr($data[$c], -1) == '"')
                    $data[$c] = substr($data[$c], 0, -1);
                $data[$c] = str_replace('""', '"', $data[$c]);

                $phones[$c] = htmlentities($data[$c], ENT_COMPAT, 'UTF-8');
            }

            if(is_array($sheets[0]))
            {
                $i = 0;
                $inn = '';
                $legal_entity = '';
                $originator = '';
                $comments = '';
                foreach($sheets[0] as $key => $val)
                {
                    if($sheets[0][$key])
                        ${$sheets[0][$key]} = check_xls_value($phones[$key]);
                    $i++;
                }
                if(count($operators)>0){
                    foreach($operators as $operator_one){
                        $MCC = $operators_all[$operator_one]["MCC"];
                        $MNC = $operators_all[$operator_one]["MNC"];
                        $type_originator = $operators_all[$operator_one]["type_originator"];
                        if(is_valid_inn($inn)){
                            if($legal_entity){
                                $query = "SELECT `id_originator` FROM `bill_register_originator` WHERE `originator`='{$originator}' AND `inn`='{$inn}' AND `id_user`='{$id_user}' AND `MCC`='{$MCC}' AND `MNC`='{$MNC}' AND `type_originator`='{$type_originator}' LIMIT 1";
                                $result = $SQL->query($query);
                                $num = $SQL->num_rows($result);
                                if($num>0){
                                    error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $global_languages["SENDER_ALLREADY_EXISTS"]);
                                } else {
                                    if(check_originator($originator)){
                                        $count_load++;
                                        $num_load += add_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $id_partner, $id_manager, $inn_client);
                                    } else {
                                        error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $global_languages["SENDER_CAN_ONLY_CONSIST"]);
                                    }
                                } 
                            } else {
                                error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $global_languages["ERROR_LEGAL_ENTITY"]);
                            }
                        } else {
                            error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $global_languages["ERROR_INN"]);
                        }
                    }
                }
            }
            $rest--;
            if($count_load%25 == 0)
                set_num_sms($id_background, 1, $rest_array, $num_load);
        }
        fclose ($fp);
        }
        set_progress_background($id_background, $rest, $num_load);
        return $num_load;
    }
    
    function error_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $error=''){
        global $SQL;
        $query = 'insert into bill_error_originator set id_user="' . $id_user . '", originator="' . $originator . '",  type_originator="' . $type_originator . '", legal_entity="' . $legal_entity . '", inn="'.$inn.'", MCC="'.$MCC.'", MNC ="'.$MNC.'", comments ="'.$comments.'", error="'.$error.'"';
        $SQL->query($query);
    }
    
    function add_registry_originator($inn, $legal_entity, $originator, $comments, $MCC, $MNC, $type_originator, $id_user, $id_partner, $id_manager, $inn_client){
        global $SQL;       
        
        $query = "SELECT `id_originator` FROM `bill_register_originator` WHERE `originator`='{$originator}' AND (`inn`!='{$inn}' OR `legal_entity`!='{$legal_entity}') LIMIT 1";
        $result = $SQL->query($query);
        $num = $SQL->num_rows($result);
        if($result and $num>0){
            $master = 0;
        } else {
            $master = 1;
        }        
        
        $query = 'insert into  `bill_register_originator` SET 
                `originator`="' . $originator . '",
                `template`="",
                `type_originator`="' . $type_originator . '",
                `legal_entity`="' . $legal_entity . '",
                `inn`="' . $inn . '",
                `inn_client`="' . $inn_client . '",
                `id_partner`="' . $id_partner . '",
                `id_user`="' . $id_user . '",
                `id_manager`="' . $id_manager . '",
                `master`="' . $master . '",
                `MCC`="' . $MCC . '",
                `MNC`="' . $MNC . '",
                `comments_user`="' . $comments . '",
                `aggregating`=""';
        $SQL->query($query);
        $id_originator = $SQL->insert_id();
        $query = 'INSERT INTO `bill_originator_status` SET `id_originator`="' . $id_originator . '",`status`="posted_for_approval",`date_from`="' . date("Y-m-d") . '"';
        $SQL->query($query);
        return 1;
    }
    
    function get_list_stats_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        $result['date_start'] = $xml['stats'][0]['a']['date_start'];
        $result['date_stop'] = $xml['stats'][0]['a']['date_stop'];
        $result['state'] = $xml['stats'][0]['a']['state'];
        $result['originator'] = $xml['stats'][0]['a']['originator'];
        $result['phone'] = $xml['stats'][0]['a']['phone'];
        $result['operator'] = $xml['stats'][0]['a']['operator'];
        $result['from_hour'] = $xml['stats'][0]['a']['from_hour'];
        $result['to_hour'] = $xml['stats'][0]['a']['to_hour'];
        $result['from_minute'] = $xml['stats'][0]['a']['from_minute'];
        $result['to_minute'] = $xml['stats'][0]['a']['to_minute'];
        return $result;
    }
    
    function send_list_stats_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user, local_time, id_partner from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            $local_time = $SQL->result($result, 0, 'local_time');
            $id_partner = $SQL->result($result, 0, 'id_partner');
            if(check_user_ip_xml($id_user))
            {
                if($id_partner)
                    $sell = 'sell_partner';
                else
                    $sell = 'sell_we';
                $state_where = $operator_where = $phone_where = $originator_where = '';
                if($xml['date_start'] and preg_match('/^(20[0-9]{2})-(0[1-9]|1[0-2])-(0[1-9]|[1-2][0-9]|3[0-1])$/', $xml['date_start'])){
                    
                } elseif($xml['date_start'] and preg_match('/(0[1-9]|[12][0-9]|3[01])\.(0[1-9]|1[012])\.(20[0-9]{2})$/', $xml['date_start'], $matches)){
                    $xml['date_start'] = $matches[3]."-".$matches[2]."-".$matches[1];
                }elseif(!$xml['date_start']){
                    $xml['date_start'] = substr(date_add_minute(date('Y-m-d H:i:s'), $local_time), 0, 10);
                } else
                    $resp['error'] = $global_languages['BAD_FORMAT_START_DATE'];
                
                if($xml['date_stop'] and preg_match('/^(20[0-9]{2})-(0[1-9]|1[0-2])-(0[1-9]|[1-2][0-9]|3[0-1])$/', $xml['date_stop'])){
                    
                } elseif($xml['date_stop'] and preg_match('/(0[1-9]|[12][0-9]|3[01])\.(0[1-9]|1[012])\.(20[0-9]{2})$/', $xml['date_stop'], $matches)){
                    $xml['date_stop'] = $matches[3]."-".$matches[2]."-".$matches[1];
                }elseif(!$xml['date_stop']){
                    $xml['date_stop'] = substr(date_add_minute(date('Y-m-d H:i:s'), $local_time), 0, 10);
                } else
                    $resp['error'] = $global_languages['BAD_FORMAT_END_DATE'];
                if(!$xml['from_hour'])
                    $xml['from_hour'] = '00';
                else {
                    $len = strlen($xml['from_hour']);
                    if($len == 1)
                        $xml['from_hour'] = "0".$xml['from_hour'];
                }
                if(!$xml['from_minute'])
                    $xml['from_minute'] = '00';
                else {
                    $len = strlen($xml['from_minute']);
                    if($len == 1)
                        $xml['from_minute'] = "0".$xml['from_minute'];
                }
                $from_second = "00";
                if(!$xml['to_hour'])
                    $xml['to_hour'] = '23';
                else {
                    $len = strlen($xml['to_hour']);
                    if($len == 1)
                        $xml['to_hour'] = "0".$xml['to_hour'];
                }
                if(!$xml['to_minute'])
                    $xml['to_minute'] = '59';
                else {
                    $len = strlen($xml['to_minute']);
                    if($len == 1)
                        $xml['to_minute'] = "0".$xml['to_minute'];
                }
                $to_second = "00";
                if($xml['to_minute']=='59')
                    $to_second = '59';
                if($xml['state'] != '')
                    $state_where = ' and ss.state="' . $state . '"';
                if($xml['operator'] != ''){
                    $query = 'select sql_cache * from sms_phone_code_title where title LIKE "' . $xml['operator'] . '" LIMIT 1';
                    $result = $SQL->query($query);
                    if($SQL->num_rows($result))
                    {
                        $MCC = $SQL->result($result, 0, 'MCC');
                        $MNC = $SQL->result($result, 0, 'MNC');
                        if($MCC)
                            $operator_where .= ' and sst.MCC="'.$MCC.'"';
                        if($MNC)
                            $operator_where .= ' and sst.MNC="'.$MNC.'"';
                    } else
                        $resp['error'] = $global_languages["OPERATOR_NOT_DEFINED"];
                }
                
                if($xml['phone'] != ''){
                    $xml['phone'] = get_replace_phone($xml['phone']);
                    $phone_where = ' and sst.phone="' . $xml['phone'] . '"';
                }
                if($xml['originator'] != '')
                    $originator_where = ' and sst.originator like "' . $xml['originator'] . '"';
                
                
                if(!$resp['error']){
                    $query = 'select ss.id_state, sst.id_sms, ss.state, ss.part_no, ss.time_change_state, ss.id_smpp_error, sst.id_name_delivery, sst.time, sst.phone, sst.num_parts, sst.originator, sst.text_sms, ss.'.$sell.' as price, ss.repayment_'.$sell.' as price_return, sst.MCC, sst.MNC, spct.title from sms_send_trafic as sst LEFT JOIN sms_state_final as ss ON(ss.id_sms=sst.id_sms) LEFT JOIN `sms_phone_code_title` AS `spct` ON ( sst.MCC = spct.MCC AND sst.MNC = spct.MNC ) where sst.id_user="' . $id_user . '" and sst.time>="' . date_add_minute($xml['date_start'] . ' ' . $xml['from_hour'] . ':' . $xml['from_minute'] . ':' . $from_second, -$local_time) . '" and sst.time<="' . date_add_minute($xml['date_stop'] . ' ' . $xml['to_hour'] . ':' . $xml['to_minute'] . ':' . $to_second, -$local_time) . '"' . $originator_where . $phone_where . $operator_where . $state_where . ' order by time desc';
                    $result = $SQL->query($query);
                    $num = $SQL->num_rows($result);
                    $resp["num_stats"] = $num;
                    if($num){
                        while($res = $SQL->fetch_assoc($result)){
                            $id = $res['id_sms'];
                            $part_no = $res['part_no'];
                            $title = $res['title'];
                            $id_name_delivery = $res['id_name_delivery'];
                            $name_delivery = get_name_delivery($id_name_delivery);
                            $time_change_state = date_add_minute($res['time_change_state'], $local_time);
                            if(!$res['state'])
                                $res['state'] = 'not_deliver';
                            $send = get_state($res['state']);
                            $time = date_add_minute($res['time'], $local_time);
                            $num_parts = $res['num_parts'];
                            $text = htmlspecialchars(get_part_sms($res['text_sms'], $part_no));
                            $error = get_param_id_smpp_error($res['id_smpp_error']);
                            $price=$res['price']-$res['price_return'];
                            $resp['stats'][] = array('id_state' => $res['id_state'], 'part_no' => $part_no, 'id_sms' => $res['id_sms'], 'operator' => $title, 'name_delivery' => $name_delivery, 'phone' => $res['phone'], 'originator' => $res['originator'], 'time_change_state' => $time_change_state, 'time' => $time, 'status' => $res['state'], 'status_translate' => $send["text"], 'text' => $text, 'price' => $price, 'num_parts' => $num_parts);
                        }
                    }
                }               
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }
    
    function resp_list_stats_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer['stats']))
            {
                foreach($answer['stats'] as $stat)
                {
                    $resp .= '<stat id_sms="' . htmlspecialchars($stat['id_sms']) . '" id_state="' . htmlspecialchars($stat['id_state']) . '" operator="' . htmlspecialchars($stat['operator']) . '" name_delivery="' . htmlspecialchars($stat['name_delivery']) . '" phone="' . htmlspecialchars($stat['phone']) . '" originator="' . htmlspecialchars($stat['originator']) . '" time_change_state="' . htmlspecialchars($stat['time_change_state']) . '" time="' . htmlspecialchars($stat['time']) . '" status="' . htmlspecialchars($stat['status']) . '" status_translate="' . htmlspecialchars($stat['status_translate']) . '" text="' . htmlspecialchars($stat['text']) . '" price="' . htmlspecialchars($stat['price']) . '" num_parts="' . htmlspecialchars($stat['num_parts']) . '" part_no="' . htmlspecialchars($stat['part_no']) . '"></stat>';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    <stats num_stats="' . htmlspecialchars($answer['num_stats']) . '">
        ' . $resp . '
    </stats>
</response>';

        }
        return $resp;
    }
    
    function send_mail_control($id_user="0")
    {
        global $SQL;
        if($id_user){
            $query = 'insert ignore into sms_control_text_send_email set id_user="' . $id_user . '"';
            $SQL->query($query);
        }
    }
    
    function get_operators_all()
    {
        $operators_all = array();
        $operators_all['mega'] = array('id'=>'mega','name'=>'Мегафон (бесплатно)','checked'=>'','MCC'=>'250','MNC'=>'2','type_originator'=>'mp');
        //$operators_all['mega_step'] = array('id'=>'mega_step','name'=>'Мегафон (платно)','checked'=>'','MCC'=>'250','MNC'=>'2','type_originator'=>'ip');
        //$operators_all['bee_step'] = array('id'=>'bee_step','name'=>'Билайн (платно)','checked'=>'','MCC'=>'250','MNC'=>'99','type_originator'=>'ip');
        $operators_all['mts'] = array('id'=>'mts','name'=>'МТС (бесплатно)','checked'=>'','MCC'=>'250','MNC'=>'1','type_originator'=>'mp');
        //$operators_all['mts_step'] = array('id'=>'mts_step','name'=>'МТС (платно)','checked'=>'','MCC'=>'250','MNC'=>'1','type_originator'=>'ip');
        //$operators_all['motiv_step'] = array('id'=>'motiv_step','name'=>'Мотив (платно)','checked'=>'','MCC'=>'250','MNC'=>'16','type_originator'=>'ip');
        $operators_all['tele2'] = array('id'=>'tele2','name'=>'Теле2 (бесплатно)','checked'=>'','MCC'=>'250','MNC'=>'20','type_originator'=>'mp');
        //$operators_all['tele2_step'] = array('id'=>'tele2_step','name'=>'Теле2 (платно)','checked'=>'','MCC'=>'250','MNC'=>'20','type_originator'=>'ip');
        return $operators_all;
    }
    
    function get_operator_register()
    {
        $operator_register = array();
        $operator_register['mp']["250:2"] = "mega";
        $operator_register['ip']["250:2"] = "mega_step";
        $operator_register['ip']["250:99"] = "bee_step";
        $operator_register['mp']["250:1"] = "mts";
        $operator_register['ip']["250:1"] = "mts_step";
        $operator_register['ip']["250:16"] = "motiv_step";
        $operator_register['mp']["250:20"] = "tele2";
        $operator_register['ip']["250:20"] = "tele2_step";        
        return $operator_register;        
        
    }
    
    function send_list_patterns_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                $query = 'select * from sms_pattern where id_user="' . $id_user . '"';
                $result = $SQL->query($query);
                $num = $SQL->num_rows($result);
                for($i=0; $i<$num; $i++)
                {
                    $id_pattern = $SQL->result($result, $i, 'id_pattern');
                    $name = $SQL->result($result, $i, 'name');
                    $pattern = $SQL->result($result, $i, 'pattern');
                    $resp[] = array('id_pattern' => $id_pattern, 'name' => $name, 'pattern' => $pattern);
                }
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }
    
    function send_patterns_from_array($xml)
    {
        global $SQL, $global_languages;
        $query = 'select id_user from sms_send_users where login="' . addslashes($xml['login']) . '" and passXML=password("' . addslashes($xml['password']) . '")';
        $result = $SQL->query($query);
        if($SQL->num_rows($result))
        {
            $id_user = $SQL->result($result, 0, 'id_user');
            if(check_user_ip_xml($id_user))
            {
                if(is_array($xml['patterns']))
                {
                    foreach($xml['patterns'] as $pattern)
                    {
                        if($pattern['id_pattern'])
                        {
                            $query = 'SELECT count(*) as count FROM sms_pattern where id_pattern="' . addslashes($pattern['id_pattern']) . '" and id_user="' . $id_user . '"';
                            $result = $SQL->query($query);
                            $res = $SQL->fetch_assoc($result);
                            if($res['count']==1)
                            {
                                $query = 'update sms_pattern set name="' . addslashes($pattern['name']) . '", pattern="' . addslashes($pattern['pattern']) . '" where id_pattern="' . addslashes($pattern['id_pattern']) . '" and id_user="' . $id_user . '"';
                                $SQL->query($query);
                                if($SQL->affected_rows())
                                    $pattern['action'] = 'edit';
                                else
                                    $pattern['action'] = 'not_edit';
                            }else
                                $pattern['action'] = 'not_found';
                        }
                        else
                        {
                            $query = 'insert into sms_pattern set name="' . addslashes($pattern['name']) . '", pattern="' . addslashes($pattern['pattern']) . '", id_user="' . $id_user . '"';
                            $SQL->query($query);
                            $pattern['id_pattern'] = $SQL->insert_id();
                            $pattern['action'] = 'insert';
                        }
                        $resp[] = array('id_pattern' => $pattern['id_pattern'], 'number_pattern' => $pattern['number_pattern'], 'action' => $pattern['action']);
                    }
                }
                if(is_array($xml['delete_patterns']))
                {
                    foreach($xml['delete_patterns'] as $pattern)
                    {
                        if($pattern['id_pattern'])
                        {
                            $query = 'delete from sms_pattern where id_pattern="' . addslashes($pattern['id_pattern']) . '" and id_user="' . $id_user . '"';
                            $SQL->query($query);
                            if($SQL->affected_rows())
                                $pattern['action'] = 'delete';
                            else
                                $pattern['action'] = 'not_found';
                        }
                        $resp[] = array('id_pattern' => $pattern['id_pattern'], 'action' => $pattern['action']);
                    }
                }
            } else
                $resp['error'] = $global_languages['INCORRECT_IP_USER'];
        }
        else
            $resp['error'] = $global_languages['INCORRECT_LOGIN_OR_PASSWORD'];
        return $resp;
    }
    
    function get_list_patterns_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        return $result;
    }
    
    function resp_list_patterns_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer))
            {
                foreach($answer as $pattern)
                {
                    $resp .= '<pattern id_pattern="' . htmlspecialchars($pattern['id_pattern']) . '" name="' . htmlspecialchars($pattern['name']) . '">' . htmlspecialchars($pattern['pattern']) . '</pattern>';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';

        }
        return $resp;
    }
    
    function get_patterns_from_xml($in)
    {
        $in = str_replace('><', '> <', $in);
        $xml = XML_in_array($in);
        if($xml == FALSE)
            return FALSE;
        $result['login'] = $xml['security'][0]['v']['login'][0]['a']['value'];
        $result['password'] = $xml['security'][0]['v']['password'][0]['a']['value'];
        if(isset($xml['patterns'][0]['v']['pattern']))
        {
           $i=0;
            foreach($xml['patterns'][0]['v']['pattern'] as $pattern)
            {
                $result['patterns'][] = array('id_pattern' => $pattern['a']['id_pattern'], 'number_pattern' => $pattern['a']['number_pattern'], 'name' => $pattern['a']['name'], 'pattern' => $pattern['v']);
            }
        }
        if(isset($xml['delete_patterns'][0]['v']['pattern']))
        {
            foreach($xml['delete_patterns'][0]['v']['pattern'] as $pattern)
            {
                $result['delete_patterns'][] = array('id_pattern' => $pattern['a']['id_pattern']);
            }
        }
        return $result;
    }
    
    function resp_patterns_from_array($answer)
    {
        if(isset($answer['error']))
            $resp = send_xml_error($answer['error']);
        else
        {
            if(is_array($answer))
            {
                foreach($answer as $pattern)
                {
                    if($pattern['number_pattern'])
                        $number_pattern = ' number_pattern="' . htmlspecialchars($pattern['number_pattern']) . '"';
                    else
                        $number_pattern = '';
                    $resp .= '<pattern id_pattern="' . htmlspecialchars($pattern['id_pattern']) . '"' . $number_pattern . '>' . $pattern['action'] . '</pattern>';
                }
            }

            $resp = '<?xml version="1.0" encoding="utf-8"?>
<response>
    ' . $resp . '
</response>';
        }
        return $resp;
    }
?>
