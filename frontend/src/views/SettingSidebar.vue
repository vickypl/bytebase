<template>
  <CommonSidebar
    :key="'setting'"
    :item-list="(settingSidebarItemList as SidebarItem[])"
    :get-item-class="getItemClass"
  />
</template>

<script lang="ts" setup>
import {
  UserCircle,
  Building,
  ShieldCheck,
  Link,
  Archive,
} from "lucide-vue-next";
import { computed, h } from "vue";
import { useI18n } from "vue-i18n";
import { useRoute } from "vue-router";
import { SidebarItem } from "@/components/CommonSidebar.vue";
import { useCurrentUserV1 } from "@/store";
import { hasSettingPagePermission } from "../utils";

const { t } = useI18n();
const route = useRoute();
const currentUserV1 = useCurrentUserV1();

const getItemClass = (path: string | undefined) => {
  const list = [];
  switch (route.name) {
    case "setting.workspace.sso.detail":
    case "setting.workspace.sso.create":
      if (path === "/setting/sso") {
        list.push("router-link-active", "bg-link-hover");
      }
      break;
    case "workspace.profile":
      if (path === "/setting/member") {
        list.push("router-link-active", "bg-link-hover");
      }
      break;
    case "setting.workspace.sql-review.create":
    case "setting.workspace.sql-review.detail":
      if (path === "/setting/sql-review") {
        list.push("router-link-active", "bg-link-hover");
      }
      break;
  }
  return list;
};

const settingSidebarItemList = computed((): SidebarItem[] => {
  return [
    {
      title: t("settings.sidebar.account"),
      icon: h(UserCircle),
      type: "div",
      children: [
        {
          title: t("settings.sidebar.profile"),
          path: "/setting/profile",
          type: "route",
        },
      ],
    },
    {
      title: t("settings.sidebar.workspace"),
      icon: h(Building),
      type: "div",
      children: [
        {
          title: t("settings.sidebar.general"),
          path: "/setting/general",
          type: "route",
        },
        {
          title: t("settings.sidebar.members"),
          path: "/setting/member",
          hide: !hasSettingPagePermission(
            "setting.workspace.member",
            currentUserV1.value.userRole
          ),
          type: "route",
        },
        {
          title: t("settings.sidebar.custom-roles"),
          path: "/setting/role",
          type: "route",
        },
        {
          title: t("settings.sidebar.subscription"),
          path: "/setting/subscription",
          type: "route",
        },
        {
          title: t("settings.sidebar.debug-log"),
          path: "/setting/debug-log",
          hide: !hasSettingPagePermission(
            "setting.workspace.debug-log",
            currentUserV1.value.userRole
          ),
          type: "route",
        },
      ],
    },
    {
      title: t("settings.sidebar.security-and-policy"),
      icon: h(ShieldCheck),
      type: "div",
      children: [
        {
          title: t("sql-review.title"),
          path: "/setting/sql-review",
          type: "route",
        },
        {
          title: t("slow-query.self"),
          path: "/setting/slow-query",
          type: "route",
        },
        {
          title: t("schema-template.self"),
          path: "/setting/schema-template",
          type: "route",
        },
        {
          title: t("custom-approval.risk.risk-center"),
          path: "/setting/risk-center",
          type: "route",
        },
        {
          title: t("custom-approval.self"),
          path: "/setting/custom-approval",
          type: "route",
        },
        {
          title: t("settings.sidebar.sensitive-data"),
          path: "/setting/sensitive-data",
          hide: !hasSettingPagePermission(
            "setting.workspace.sensitive-data",
            currentUserV1.value.userRole
          ),
          type: "route",
        },
        {
          title: t("settings.sidebar.access-control"),
          path: "/setting/access-control",
          hide: !hasSettingPagePermission(
            "setting.workspace.access-control",
            currentUserV1.value.userRole
          ),
          type: "route",
        },
        {
          title: t("settings.sidebar.audit-log"),
          path: "/setting/audit-log",
          hide: !hasSettingPagePermission(
            "setting.workspace.audit-log",
            currentUserV1.value.userRole
          ),
          type: "route",
        },
      ],
    },
    {
      title: t("settings.sidebar.integration"),
      icon: h(Link),
      type: "div",
      children: [
        {
          title: t("settings.sidebar.gitops"),
          path: "/setting/gitops",
          hide: !hasSettingPagePermission(
            "setting.workspace.gitops",
            currentUserV1.value.userRole
          ),
          type: "route",
        },
        {
          title: t("settings.sidebar.sso"),
          path: "/setting/sso",
          hide: !hasSettingPagePermission(
            "setting.workspace.sso",
            currentUserV1.value.userRole
          ),
          type: "route",
        },
        {
          title: t("settings.sidebar.mail-delivery"),
          path: "/setting/mail-delivery",
          hide: !hasSettingPagePermission(
            "setting.workspace.mail-delivery",
            currentUserV1.value.userRole
          ),
          type: "route",
        },
      ],
    },
    {
      title: t("common.archived"),
      icon: h(Archive),
      path: "/setting/archive",
      type: "route",
    },
  ];
});
</script>
